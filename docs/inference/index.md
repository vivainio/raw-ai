---
icon: lucide/cpu
---

# Theory: Inference

The [Agent Dialogue](../agent-dialogue/index.md) chapter describes calling a
tool like this: "the model predicts a JSON object — a `tool_use` block —
and stops." That line undersells how generic it is. The model doesn't have
a separate mode for "writing a tool call" versus "writing a sentence." It
does exactly one thing, always: given everything so far, produce the next
token. A sentence, a JSON block, a line of Python — all of it comes out of
the same loop, one token at a time, run until something tells it to stop.
This chapter is about that loop.

## Tokens, not words

Text going in gets cut into tokens before the model ever sees it — chunks
from a fixed vocabulary, not words or characters. As a rough rule of thumb
for English prose, one token is about four characters — good enough for
estimating context usage, not for anything exact. Every number quoted for
context window size, pricing, or "tokens used" is counted in these units.
Why the split lands where it does, and why that same rule of thumb falls
apart for code and non-English text, is the subject of the
[Tokenization](../tokenization/index.md) chapter — the loop this chapter
covers starts one step later, with a sequence of token IDs already in
hand.

## One forward pass, one token

Feed the model's network a sequence of token IDs and it produces, for the
last position, a score for every entry in the vocabulary. Those raw scores
are **logits**, not probabilities. Turning them into an actual next token
is a separate, comparatively cheap step: scale the logits by a temperature,
convert them into probabilities with softmax, optionally cut the field down
to the top-k candidates or the smallest set whose probabilities reach the
top-p threshold, then sample. Higher temperature flattens the distribution;
lower temperature sharpens it toward the top candidates. What APIs call
temperature 0 normally means greedy decoding: take the highest-scoring
token instead of sampling. That removes sampling randomness, but doesn't
make the answer correct or guarantee bit-for-bit reproducibility across
different inference systems.

Whatever token comes out gets appended to the sequence. Conceptually, the
model then processes everything so far to produce the next one; in a real
inference server, the KV cache described below avoids recomputing the whole
prefix. The generation loop is:

```
tokens = tokenize(prompt)
cache = prefill(tokens)
while not stopped:
    logits = next_logits(cache)
    next_token = sample(logits)     # temperature / top-p / top-k
    tokens.append(next_token)
    stopped = next_token == EOS or hit_limit or matched_stop_sequence
    cache = decode_one(next_token, cache)
detokenize(tokens)
```

There are several ways out of the loop. The model can emit an
end-of-sequence token; the generated tokens can complete a configured stop
sequence; or the server can enforce a length or time limit. Structured
formats may also use model- or API-specific boundary tokens. The model
still generates all of these autoregressively, but the server or harness
decides which boundaries end a response and whether the accumulated tokens
represent text to display, arguments to parse, or a tool call to execute.

## Prefill: one pass over the whole prompt

Before anything gets generated, the model has to run the prompt through
the network at least once — turn "everything so far" into the internal
state needed to predict the first output token. Every position in the
prompt already has a known token ID, so this step, called **prefill**,
isn't sequential the way generation is: for each layer, the query, key,
and value vectors for every position get computed in the same batched
matrix multiply, and attention for all positions runs in parallel too. A
causal mask — not a processing order — is what stops position *i* from
attending to positions after it: those future positions are still computed
alongside everything else, but their attention scores are masked out before
softmax. One pass through the network, over the whole prompt at once, is
enough.

That pass produces two things, not one. The obvious one is the logits
needed to sample the first output token. The other is a byproduct: a key
and value vector for every prompt position, at every layer, computed along
the way. Those vectors are exactly what the KV cache (next section) starts
out holding — prefill doesn't just get the first token ready, it populates
the cache that every subsequent decode step is going to read from.

Because prefill can do large matrix multiplications across many positions
at once, it usually uses the accelerator's compute units more efficiently
than single-token decode and is often **compute-bound**. That is a common
operating regime, not a rule: prompt length, batch size, attention
implementation, model, and hardware can shift the bottleneck toward memory
movement. Prefill is usually the main model-side component of **time to
first token**, since nothing gets sampled until it finishes, but observed
latency can also include queueing, cache lookup, scheduling, tokenization,
and network time.

## Why the prompt is fast and the answer is slow

Generating the reply can't use the same trick. Token 50 depends on token 49
actually having been sampled first — there's no way to compute it in
advance. So generation, called **decode**, runs one sequential step per
output token. With one request or a small batch, each step does
comparatively little arithmetic while reading a large amount of model and
cache data from accelerator memory, so decode is typically
**memory-bandwidth-bound**. Larger batches reuse the loaded model weights
across more tokens and can move the bottleneck back toward compute.

The unavoidable part is the sequence: a 3,000-token answer requires 3,000
decode steps, however efficiently requests are batched. That is why output
length usually dominates generation latency, while a prompt of the same
length can be processed in parallel during prefill. It does not make input
free—very long prompts still require substantial work, and API pricing may
value input and output differently. Extended-thinking or hidden reasoning
adds latency for the same reason visible output does: every additional
reasoning token is another decode step, even if the interface never shows
it.

## The KV cache

Attention — the mechanism that lets a token's computation depend on every
earlier token, covered in [Transformer
Architecture](../transformer-architecture/index.md#self-attention-query-key-value)
— needs a key and value vector for every prior position at every layer.
Recomputing those from scratch at each new decode step would
mean step *t* redoes work already done at steps 1 through *t-1*. Even the
per-token projections and MLPs would add up quadratically across the
growing sequence, and repeatedly running full attention would be more
expensive still. Instead, every key and value vector gets computed once
and kept in accelerator memory — the **KV cache**. A decode step runs the
new token through every layer, adding its new keys and values to the cache
and attending against the keys and values already there. Attention still
has an increasing amount of cached context to read, but the earlier token
representations are not rebuilt from scratch.

This connects two things that otherwise look unrelated. First, context
window limits are partly a memory problem, not just an attention-quality
one: the cache grows with every token in the conversation and has to fit
alongside the model and every other concurrent request's state. Model
configuration and the cost of attending over that context impose limits
too.

Second, "prompt caching" as a billed API feature can reuse intermediate
state for a matching prefix across requests. If a request's first N tokens
— a large system prompt or a block of tool definitions — are identical to
a previously processed prefix, the serving system may reuse cached KV data
or equivalent intermediate blocks instead of recomputing them. Providers
differ in how they identify, store, and distribute those blocks; the useful
mental model is a cache hit on prefix computation, not a cache of the final
answer.

## Streaming is just not buffering

Because decode already produces exactly one token at a time, sending each
one to the client as soon as it's sampled — instead of waiting for the
whole reply and sending it in one response — doesn't change the model's
core computation. The serving layer still has to detokenize, serialize,
buffer, and transmit the output, usually in chunks containing one or more
tokens over a chunked HTTP response or server-sent events. A chat UI then
renders the resulting text, which is why it may appear character by
character even though characters are not what the model emits.

Streaming exposes decode progress, but not perfectly. A pause or burst can
come from batching, server scheduling, network buffering, speculative
decoding, a tool boundary, hidden reasoning, or tokens that produce no
visible text. The display alone cannot tell you which one occurred.

## Batching happens across requests, not just within one

For a small decode batch, the accelerator may have much more arithmetic
capacity than one next-token step can use. Inference servers commonly
improve utilization by decoding many users' in-flight requests together:
sequence A's next token, sequence B's next token, and sequence C's next
token are computed in one batched operation. The model-weight reads are
amortized across the batch even though each request has separate tokens and
KV-cache data.

In **continuous batching** (or **in-flight batching**), requests join and
leave batches as they start and finish. Server load and scheduling can
therefore change a request's latency even when its prompt stays identical.
This is different from a provider's discounted "batch API" for
non-interactive jobs, which trades response time for price. Continuous
batching is a common internal serving technique, not something the client
opts into and not a guarantee about every provider's implementation.

## What this buys back

These are common mechanics of transformer-based autoregressive models, not
features of one vendor, although vocabularies, cache layouts, batching, and
structured-output protocols vary between serving systems. They explain why
prose, code, and tool-call arguments can all emerge from the same
next-token loop; why output length usually has an outsized effect on
latency; and why an identical system-prompt or tool-definitions prefix is
exactly the kind of computation a prompt cache can reuse. The precise cost
balance still depends on input length, output length, cache hits, batching,
hardware, and the provider's pricing.

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
last position, a score for every entry in the vocabulary — how likely each
one is to come next. Turning those scores into an actual next token is a
separate, comparatively cheap step: scale the scores by a temperature
(higher temperature flattens the distribution, more randomness; lower
sharpens it toward the top candidates), optionally cut the field down to
the top-k or top-p mass, then sample. Temperature 0 skips the randomness
entirely and always takes the single highest-scoring token — deterministic,
but not thereby "correct."

Whatever token comes out gets appended to the sequence, and the whole
sequence — original prompt plus every token generated so far — gets run
through the model again to produce the next one. That's the entire
generation loop:

```
tokens = tokenize(prompt)
while not stopped:
    scores = model(tokens)          # one score per vocabulary entry, for the last position
    next_token = sample(scores)     # temperature / top-p / top-k, then pick one
    tokens.append(next_token)
    stopped = next_token == EOS or hit_max_tokens or matched_stop_sequence
detokenize(tokens)
```

A stop is just a token — an end-of-sequence marker, or one that matches a
configured stop sequence. Nothing distinguishes "the model finished its
answer" from "the model finished a `tool_use` block" at this layer; both
are the loop reaching a token that ends it. What that accumulated run of
tokens *means* — plain text to show you, or JSON to parse into a tool
call — is a decision the harness makes afterward, not something the
decoding loop is aware of.

## Prefill: one pass over the whole prompt

Before anything gets generated, the model has to run the prompt through
the network at least once — turn "everything so far" into the internal
state needed to predict the first output token. Every position in the
prompt already has a known token ID, so this step, called **prefill**,
isn't sequential the way generation is: for each layer, the query, key,
and value vectors for every position get computed in the same batched
matrix multiply, and attention for all positions runs in parallel too. A
causal mask — not a processing order — is what stops position *i* from
attending to positions after it: the future positions are still computed
alongside everything else, then zeroed out of position *i*'s weighted sum.
One pass through the network, over the whole prompt at once, is enough.

That pass produces two things, not one. The obvious one is the logits
needed to sample the first output token. The other is a byproduct: a key
and value vector for every prompt position, at every layer, computed along
the way. Those vectors are exactly what the KV cache (next section) starts
out holding — prefill doesn't just get the first token ready, it populates
the cache that every subsequent decode step is going to read from.

Because prefill is doing large, batched matrix multiplications across
every position at once, it's **compute-bound**: its cost tracks available
GPU FLOPs and scales with how long the prompt is, not with how the GPU's
memory bandwidth happens to be doing. That's also why "time to first
token" — the latency a spinner is covering for, before any text appears —
is essentially "how long prefill took": nothing gets sampled until this
one pass over the whole prompt finishes.

## Why the prompt is fast and the answer is slow

Generating the reply can't use the same trick. Token 50 depends on token 49
actually having been sampled first — there's no way to compute it in
advance. So generation, called **decode**, runs one sequential step per
output token, each step doing comparatively little compute but still
paying the cost of moving the model's weights through memory — it's
**memory-bound**, the opposite bottleneck from prefill's. That asymmetry
is the mechanical reason a 3,000-token prompt with a 50-token reply is fast
and cheap, while a 200-token prompt asking for a 3,000-token answer is slow
and expensive in comparison — the bottleneck was never the input, it's the
number of sequential decode steps the output requires. It's also why
extended-thinking / reasoning output inflates latency more than an
equivalent amount of visible reply text would: it's still one decode step
per token, just tokens you don't see.

## The KV cache

Attention — the mechanism that lets a token's computation depend on every
earlier token — needs a key and value vector for every prior position at
every layer. Recomputing those from scratch at each new decode step would
mean step *t* redoes work already done at steps 1 through *t-1*, making
total cost scale quadratically with output length. Instead, every key and
value vector gets computed once and kept in GPU memory — the **KV
cache** — so a decode step only computes the query, key, and value for the
one new token and attends against everything already cached.

This is the actual mechanism behind two things that otherwise look
unrelated. First, context window limits are partly a memory problem, not
just an attention-quality one: the cache grows with every token in the
conversation and has to fit in GPU memory alongside the model itself and
every other concurrent request's cache. Second, "prompt caching" as a
billed API feature is this same cache, reused across requests: if a
request's first N tokens — a large system prompt, a big block of tool
definitions — are identical to a prior request's, the KV vectors computed
for those tokens during that prior prefill can be reused instead of
recomputed. It's an infrastructure-level cache hit on already-done matrix
multiplications, not a database lookup keyed on the request.

## Streaming is just not buffering

Because decode already produces exactly one token at a time, sending each
one to the client as soon as it's sampled — instead of waiting for the
whole reply and sending it in one response — costs nothing extra to
implement at the model layer. "Streaming" isn't a UX feature bolted on
after the fact; it's what you get by default if you don't buffer. What you
see scroll past character-by-character in a chat UI is the decode loop's
own pace, exposed directly, usually over a chunked HTTP response or
server-sent events. The pauses you sometimes notice mid-stream (a long gap
before a tool call, then a burst of text) are the loop hitting a token
sequence that happens to render nothing visible, not the model "thinking."

## Batching happens across requests, not just within one

Decode's per-step compute is small relative to the cost of moving weights
and KV cache through memory, which means a GPU sitting there decoding one
user's next token at a time is mostly idle. Inference servers exploit this
by running the same decode step for many different users' in-flight
requests together as one batched operation — sequence A's next token,
sequence B's next token, and sequence C's next token, computed in the same
pass because the memory movement is shared across them even though the
tokens are unrelated. Requests join and leave this batch continuously as
they start and finish (hence "continuous batching" / "in-flight
batching"), which is why the same request can come back faster or slower
depending on how much concurrent load the server happens to be handling —
nothing about your prompt changed, the batch it landed in did. This is a
different thing from a provider's discounted "batch API" for
non-interactive workloads, which trades latency for price; continuous
batching happens under every live, interactive request, invisible to the
client, whether you asked for it or not.

## What this buys back

None of this is specific to any one vendor — it's the shape of every
transformer-based autoregressive model currently deployed at scale, Claude
included, even though the exact vocabulary, cache layout, and batching
implementation are internal to whoever runs the servers. What it explains,
concretely, is why the rest of this book's mechanics look the way they
do: why a `tool_use` block is indistinguishable from prose at the
decoding layer until something parses it, why output length dominates
latency and cost far more than input length, and why a system prompt or
tool-definitions block that stays identical across turns is exactly the
prefix a prompt cache is built to reuse.

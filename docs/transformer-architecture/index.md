---
icon: lucide/layers
---

# Theory: Transformer Architecture

The [Tokenization](../tokenization/index.md) chapter ends with a sequence of
token IDs. The [Inference](../inference/index.md) chapter picks up one step
later, treating "run the tokens through the model" as a black box that
produces scores. This chapter is what's inside that box — kept light, on
purpose: the goal here isn't to derive the architecture, it's to give the
handful of terms you'll see in every model card, paper abstract, and
"how LLMs work" diagram an actual referent, so they stop being vocabulary
you nod along to. The focus is the decoder-only, autoregressive transformer
behind most text-generating LLMs.

## Embeddings: a token ID is not a vector, yet

A token ID is just an integer — an index into the vocabulary. The first
thing the network does is turn each one into a vector of a few thousand
numbers, via a lookup table learned during training: row *n* of that table
is token *n*'s **embedding**. Two tokens the model treats similarly end up
with similar vectors, but "similar" here isn't hand-designed — it falls out
of training the same way BPE's merges fell out of corpus statistics, not
out of anyone deciding what a token should mean.

That per-token vector is all the network has to work with going in. Every
other term below describes a transformation applied to a stack of these
vectors, one per position in the sequence.

Notice what the embedding table doesn't know: position. Row *n* is keyed on
token identity alone, so the same token occupying position 3 and position
300 in the sequence starts out as the exact same vector. Whatever tells the
network where a token sits in the sequence has to come from somewhere
else — which is the next section.

## Positional encoding: order isn't free

Attention (covered under Self-attention below) works by comparing token
representations; those comparisons don't by themselves say whether two
tokens are one position or a hundred positions apart. A decoder's causal
mask tells it which tokens came earlier, but the model still needs position
information to represent order and distance properly. That information gets
injected explicitly, and *where* it gets injected depends on the mechanism.

Older models added a fixed or learned position vector directly to each
token embedding before the first layer. Most current frontier models use
**RoPE** (rotary position embedding) instead: at every layer, it rotates
the query and key vectors derived from the current token representations
by angles that depend on position. RoPE leaves the initial embedding lookup
unchanged, but its effect flows into the position-dependent representations
produced by attention. Different mechanism, same job: giving attention a
usable signal about where tokens occur in the sequence.

## The transformer block, stacked N times

Everything from here on happens inside one repeated unit (technical name:
**layer**, or **block**), and a model is just this same unit stacked over
and over — dozens of times, for current frontier models. Two model sizes
in the same family, like "8B" and "70B," usually differ mainly in how many
times this unit repeats and how big the vectors flowing through it are,
not in what the unit itself does. Each repetition does two main things, in
order:

1. Let every token look back and pull in relevant information from itself
   and earlier tokens (technical name: **self-attention**).
2. Do some private processing on each token's vector by itself, with no
   looking around at all (technical name: **feed-forward network**, or
   **MLP**).

Each step sits on a **residual connection** that adds the step's output to
what came in, preserving a direct path through the stack. **Layer
normalization** also keeps the representations at a manageable scale. The
exact ordering varies: the original transformer normalized after each
residual addition, while most current LLMs normalize before each attention
or MLP step. Stack enough of these blocks and you have the network.

## Self-attention: query, key, value

This is the "look around" step, and it's also the reason the whole
architecture is called a *transformer* in the first place — from the 2017
paper that introduced it, *Attention Is All You Need*.

For every token, the network builds three things out of its current
representation: something describing what it's looking for, something
describing what it has to offer as a label, and something describing what
it has to offer as actual content (technical names, respectively:
**query**, **key**, **value**). Each token compares its query against the keys of
itself and every earlier token; a **causal mask** prevents it from looking
at later tokens. Those comparisons become relative importance weights
(via **softmax**), which are used to blend the corresponding values. A
strong match between one token's query and another token's key gives that
other token's value more weight. There is no fixed nearby-tokens-only
window in standard full attention, just a learned sense of which earlier
tokens matter to which.

The network doesn't do this once — it runs several independent copies of
this comparison side by side, each with its own separate "looking
for / label / content," then combines the results (technical name:
**multi-head attention**). Nothing designs what each copy ends up good
at, but in practice they specialize anyway, purely as a side effect of
training — one might end up tracking which earlier token a pronoun refers
to, another might track nearby grammar. Whatever gets computed here as
each token's "label" and "content" is exactly what the [Inference
chapter's](../inference/index.md#the-kv-cache) KV cache stores — one pair
per token, per attention head, per layer (with implementation details that
can reduce how many distinct key/value heads are stored).

## Feed-forward (MLP): the other sub-layer

After the "look around" step blends in context from earlier tokens, each
token's resulting vector goes through a second step done entirely on its
own — no comparing to other tokens at all, just transforming that one
vector (technical name: **feed-forward network**, or **MLP**). This step
usually holds the majority of a model's total learned weights, and when a
model family scales up, this is often the part that grows the most. If the
first step decides which other tokens matter, this second step is where
most of the actual per-token transformation happens.

## Mixture of experts: not every parameter runs for every token

In a dense transformer, each layer has one MLP that processes every token.
A **mixture-of-experts** model replaces that MLP with several
separate MLPs, called **experts**, plus a small learned **router** that
chooses which expert or experts should process each token at that layer.
Despite the name, an expert is not usually a complete model or a
hand-assigned specialist in subjects like code or medicine — it is just
one alternative set of feed-forward weights, and any specialization
emerges during training.

Only the selected experts run for a given token. This lets the model have
many more parameters in total without using all of them for every token:
for example, a layer might contain eight experts while routing each token
through only two, though the numbers vary by architecture. That is why MoE
model specifications often publish both **total parameters** (all model
weights, including every expert) and **active parameters** (the shared
weights plus the selected experts used for one token). Total parameters
describe the model's overall capacity and storage requirements; active
parameters are more closely related to the computation required during
inference. The tradeoff is extra routing and communication complexity,
especially when the experts are spread across multiple accelerators.

## Residuals and layer norm: why depth doesn't collapse

Stacking dozens of transformations makes optimization difficult: every
layer has to preserve useful information while gradients travel through
the entire stack during training. A **residual connection** (or **skip
connection**) makes each sub-layer learn a change to its input rather than
a complete replacement: its output gets added back to the input, leaving a
direct path through the network. **Layer normalization** rescales each
token's representation before or after a sub-layer, depending on the
architecture, which helps keep training stable. Together these mechanisms
make very deep transformer stacks practical to optimize.

## From last layer to logits

During inference, the representation at the final position is the one
needed to predict the next token. An output projection converts that vector
into one number per vocabulary entry; many models share this projection's
weights with the input embedding table, though they do not have to. Those
numbers are the raw next-token scores (technical name: **logits**) —
exactly the "score for every vocabulary entry" that the [Inference
chapter's](../inference/index.md) sampling step (temperature, top-p,
top-k) turns into an actual chosen token. During training, the same
projection is normally applied at every position so the model can learn
all next-token predictions in parallel. Vocabulary lookup in, one score
per vocabulary entry out — everything in between is what this chapter has
been describing.

## Terms, in one line each

- **Embedding** — learned vector representation of one token ID
- **Positional encoding / RoPE** — how position gets injected, since
  attention alone is order-blind
- **Layer / block** — one repetition of attention + feed-forward; a model
  is N of these stacked
- **Self-attention** — query/key/value mechanism mixing information across
  positions
- **Multi-head attention** — several attention computations run in
  parallel with independent learned projections
- **KV cache** — the stored key/value vectors attention produces per
  position per layer (see [Inference](../inference/index.md#the-kv-cache))
- **Feed-forward / MLP** — per-position transformation, no cross-position
  mixing, usually most of the parameter count
- **Mixture of experts (MoE)** — several alternative MLPs plus a learned
  router that activates only a small subset for each token
- **Residual connection** — sub-layer output added to its input, not
  replacing it
- **Layer normalization** — rescales each token's representation to help
  stabilize a deep stack
- **Logits** — the final per-vocabulary-entry scores fed into sampling
- **Parameter count** ("7B", "70B") — total learned weights across every
  embedding, projection, and feed-forward matrix in the stack

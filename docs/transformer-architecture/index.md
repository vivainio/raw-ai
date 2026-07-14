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
you nod along to.

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

Attention (next section) computes, for each position, a weighted mix of
every other position's vectors — and that computation is symmetric with
respect to order. Left unmodified, the model would see "dog bites man" and
"man bites dog" as the same bag of tokens. So position information gets
injected explicitly, and *where* it gets injected depends on the mechanism.
Older models added a fixed or learned position vector directly onto the
embedding, before the first layer — so the embedding itself became
position-dependent right there, by having something else summed into it.
Most current frontier models use **RoPE** (rotary position embedding)
instead, which never touches the embedding at all: it rotates the query and
key vectors — derived from the embedding inside attention, at every layer —
by an angle that depends on position. Different mechanism, same job, but
under RoPE the token embedding itself stays position-agnostic all the way
through; only its downstream projections carry position information.
Either way, without this step, position is information the model never
receives.

## The transformer block, stacked N times

Everything from here on happens inside a **layer** (also called a
**block**), and a model is a stack of identical ones — "70B" and "8B"
models of the same family often differ mainly in how many layers are
stacked and how wide each vector is, not in what each layer does. Each
layer has two sub-layers, applied in sequence:

1. **Self-attention** — lets each position mix in information from other
   positions.
2. **Feed-forward (MLP)** — transforms each position's vector on its own,
   no mixing across positions.

Both sub-layers are wrapped the same way: the sub-layer's output is added
back to its input (a **residual connection**) and the result is rescaled
(**layer normalization**) before moving on. Stack enough of these — dozens,
for current frontier models — and you have the network.

## Self-attention: query, key, value

This is the mechanism the [Inference](../inference/index.md#the-kv-cache)
chapter's KV cache is caching, and it's the subject of the paper that gave
the whole architecture its name — Vaswani et al.'s 2017 *Attention Is All
You Need*. Mechanically, it's three learned projections of every position's
vector:

- **Query** — what this position is looking for
- **Key** — what every position (including itself) offers, as a label
- **Value** — what every position offers, as content

For a given position, its query vector is dotted against every position's
key vector, producing one score per position; those scores go through a
softmax (turning them into positive weights that sum to 1); the position's
new vector is the weighted sum of every position's value vector, using
those weights. High score between a query and a key means "attend to this
position strongly." That's the entire mechanism — no recurrence, no fixed
window, just a learned notion of relevance between every pair of
positions.

**Multi-head** attention runs several independent copies of this (each
with its own learned Q/K/V projections) in parallel, then concatenates the
results. Nothing forces the heads to specialize in any particular way, but
in practice different heads end up picking up on different kinds of
relationships — a head can end up tracking adjacent-word syntax while
another tracks a pronoun back to what it refers to several sentences
earlier — as a byproduct of training, not a design choice. This is also
why the K and V vectors the Inference chapter's KV cache stores exist in
the first place: they're the output of the key and value projections
described here, one pair per position per layer.

## Feed-forward (MLP): the other sub-layer

After attention mixes information across positions, the feed-forward
sub-layer processes each position's resulting vector independently — the
same two-layer transformation applied identically to every position, with
no position ever looking at another one here. This sub-layer usually holds
the majority of a model's parameters; scaling a model up often means
widening this part more than any other. Where attention answers "which
other positions matter," the feed-forward block is where most of the
actual per-token transformation happens.

## Residuals and layer norm: why depth doesn't collapse

Stacking dozens of layers naively — each one replacing its input outright —
makes models effectively untrainable; gradients either vanish or explode
propagating back through that many steps. Two fixes, applied around both
sub-layers: the sub-layer's output is *added* to its input rather than
replacing it (so there's always a direct path for gradient — and for
information — to skip the sub-layer entirely), and layer normalization
rescales activations back to a consistent range before the next sub-layer
sees them. Neither changes what the network can compute; both are why a
network this deep can be trained at all.

## From last layer to logits

After the final layer, only the last position's vector matters for
predicting the next token (every earlier position's final vector goes
unused — its work was done as a key/value source for the positions after
it). That vector gets projected back to vocabulary size — the reverse of
the embedding step, and often literally the same weight matrix, transposed
— producing one number per vocabulary entry. Those numbers are the
**logits**: exactly the "score for every entry in the vocabulary" the
Inference chapter's sampling step (temperature, top-p, top-k) takes as
input. Embedding in, logits out — everything in this chapter is what
happens in between.

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
- **Residual connection** — sub-layer output added to its input, not
  replacing it
- **Layer normalization** — rescales activations to a consistent range
  between sub-layers
- **Logits** — the final per-vocabulary-entry scores fed into sampling
- **Parameter count** ("7B", "70B") — total learned weights across every
  embedding, projection, and feed-forward matrix in the stack

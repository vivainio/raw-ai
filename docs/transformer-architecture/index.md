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

Attention (covered under Self-attention below) computes, for each position, a weighted mix of
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

Everything from here on happens inside one repeated unit (technical name:
**layer**, or **block**), and a model is just this same unit stacked over
and over — dozens of times, for current frontier models. Two model sizes
in the same family, like "8B" and "70B," usually differ mainly in how many
times this unit repeats and how big the vectors flowing through it are,
not in what the unit itself does. Each repetition does two things, in
order:

1. Let every word look around and pull in relevant information from the
   other words (technical name: **self-attention**).
2. Do some private processing on each word's vector by itself, with no
   looking around at all (technical name: **feed-forward network**, or
   **MLP**).

After each of those two steps, the unit also keeps a copy of what went in
and adds it back onto what comes out, so information doesn't get lost
passing through so many repetitions (technical name: **residual
connection**), and nudges the numbers back into a sane, consistent range
before moving on (technical name: **layer normalization**). Stack enough
of these repetitions and you have the network.

## Self-attention: query, key, value

This is the "look around" step, and it's also the reason the whole
architecture is called a *transformer* in the first place — from the 2017
paper that introduced it, *Attention Is All You Need*.

For every word, the network builds three things out of that word's
vector: something describing what it's looking for, something describing
what it has to offer as a label, and something describing what it has to
offer as actual content (technical names, respectively: **query**,
**key**, **value**). Every word then compares what it's looking for
against every other word's label, turns those comparisons into a set of
relative importance scores (technical name: **softmax**), and blends in
the content from every other word weighted by those scores. A strong
match between one word's "looking for" and another word's "label" means
that other word's content gets weighted heavily. That's the whole
mechanism — no memory of past sentences, no fixed nearby-words-only
window, just a learned sense of which words matter to which.

The network doesn't do this once — it runs several independent copies of
this comparison side by side, each with its own separate "looking
for / label / content," then combines the results (technical name:
**multi-head attention**). Nothing designs what each copy ends up good
at, but in practice they specialize anyway, purely as a side effect of
training — one might end up tracking which earlier word a pronoun refers
to, another might track nearby grammar. Whatever gets computed here as
each word's "label" and "content" is exactly what the [Inference
chapter's](../inference/index.md#the-kv-cache) KV cache stores — one pair
per word, per repetition of this whole block.

## Feed-forward (MLP): the other sub-layer

After the "look around" step blends in context from other words, each
word's resulting vector goes through a second step done entirely on its
own — no comparing to other words at all, just reshaping that one vector
(technical name: **feed-forward network**, or **MLP**). This step usually
holds the majority of a model's total learned weights, and when a model
family scales up, this is often the part that grows the most. If the
first step decides which other words matter, this second step is where
most of the actual per-word transformation happens.

## Residuals and layer norm: why depth doesn't collapse

None of this works if dozens of these two-step repetitions are stacked
naively, each one throwing away and fully replacing what came in — the
numbers involved in training blow up or shrink to nothing before they can
be adjusted properly. Two small fixes prevent that, applied after both
steps in every repetition: instead of discarding the input, it gets added
onto the new output (technical name: **residual connection**, or **skip
connection**), leaving a direct, untouched path all the way through; and
the resulting numbers get rescaled to a consistent range before continuing
(technical name: **layer normalization**). Neither trick changes what the
network is capable of computing — they only make it possible to actually
train something this deep in the first place.

## From last layer to logits

After the last repetition, only the very last word's vector actually
matters — that's the one being asked to predict what comes next (every
earlier word's final vector already did its job, feeding its label and
content into later words along the way). That last vector gets converted
from "one big vector" back into "one number per vocabulary entry" —
reversing the very first embedding step, often using literally the same
lookup table, just applied backwards. Those numbers are the raw
next-token scores (technical name: **logits**) — exactly the "score for
every vocabulary entry" that the [Inference
chapter's](../inference/index.md) sampling step (temperature, top-p,
top-k) turns into an actual chosen word. Vocabulary lookup in, one score
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
- **Residual connection** — sub-layer output added to its input, not
  replacing it
- **Layer normalization** — rescales activations to a consistent range
  between sub-layers
- **Logits** — the final per-vocabulary-entry scores fed into sampling
- **Parameter count** ("7B", "70B") — total learned weights across every
  embedding, projection, and feed-forward matrix in the stack

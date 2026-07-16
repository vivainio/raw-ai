---
icon: lucide/image
---

# Models: Image Generation and Recognition

"Multimodal" gets used as if it names one capability. It names two,
built on different machinery and evaluated by different means: a model
that reads an image and answers questions about it, and a model that
produces one from a text prompt. A single vendor often ships both under
the same product name, which makes it easy to assume they're the same
system running in two directions. They aren't. This chapter treats them
separately, then comes back to where the two tracks are starting to
merge.

## Recognition: how a vision model reads an image

An image doesn't enter the model as pixels. It's cut into a grid of
fixed-size patches, each patch run through a vision encoder — typically
a Vision Transformer (ViT) — that turns it into an embedding, the same
kind of vector the [tokenization](../tokenization/index.md) chapter
described for text. Those patch embeddings get projected into the same
space as text token embeddings and dropped into the model's input
sequence alongside the prompt, which is what lets a single transformer
reason over both without a separate "vision module" bolted on the side.

That patching has a direct, billable consequence: a high-resolution
image, or a vendor's tiling scheme that splits a large image into
several sub-images to preserve detail, produces more patches — more
tokens — than a small or low-resolution one. Image input is priced and
counted in the same token budget as text, and the conversion rate is
usually documented per vendor rather than being a fixed constant across
the industry. An image dropped into a long conversation can quietly
consume a large share of the context window in a way a wall of text of
comparable "size" wouldn't.

## What recognition is reliably good and bad at

Frontier vision models are strong, unsurprisingly, at the things a
transformer trained on internet-scale image-text pairs would be
expected to be strong at: describing a scene, reading printed or
handwritten text, answering questions about a chart or a screenshot,
classifying what kind of document something is. OCR quality on clean
printed text is close to a solved problem at this point.

Three failure modes show up consistently enough to plan around rather
than treat as occasional noise:

- **Counting.** Ask a model how many objects are in an image and the
  answer degrades fast past a small number — the model is pattern-
  matching on visual density, not enumerating discrete items the way a
  detection algorithm would.
- **Precise spatial localization.** A model can usually say roughly
  where something is; asking for an exact bounding box or pixel
  coordinate is asking it to output a precise number it wasn't trained
  to produce precisely. Some vendors fine-tune specifically for
  grounding and report separate benchmarks for it — worth checking
  whether a model claims this before relying on coordinates it returns.
- **Fine detail at low salience.** Small text in the corner of a large
  image, a barely-visible watermark, a subtle inconsistency between two
  near-identical images — anything that isn't the visually dominant
  subject of the patch grid is easy for the model to miss or
  hallucinate over rather than report as unseen.

None of these are edge cases so much as the direct shadow of how patch-
based encoding works: the model sees a grid of averaged visual
information, not a lossless copy of the image, and it answers
questions the way it answers any other prompt — with the statistically
plausible response, not a measurement.

## Generation: two different architectures

Where recognition converts an image into tokens for a transformer to
read, generation goes one of two ways, and they don't share machinery.

**Diffusion models** — Stable Diffusion, Flux, Midjourney's underlying
approach — start from random noise and run a series of steps that
predict and subtract a bit of that noise at each step, guided by a text
encoding (commonly from a CLIP-style model or a separate language
model) fed in via cross-attention at every step. Most production
diffusion models do this denoising in a compressed latent space rather
than on raw pixels — a separate encoder compresses the image down and a
decoder expands the final latent back into pixels — which is most of
why modern diffusion generation is fast enough to be interactive rather
than something to leave running for minutes.

**Autoregressive image models** treat the image the same way a language
model treats text: as a sequence of discrete tokens, predicted one at a
time, left to right, conditioned on everything generated so far —
including the surrounding text in the same conversation. This is the
approach behind native image generation in models like GPT-image and
Gemini's image output, where the same transformer that handles the
conversation also produces the image tokens, rather than routing the
request out to a separate diffusion pipeline.

## Why the architecture choice matters in practice

Diffusion's strength is raw image quality and speed for a
self-contained generation task — text-to-image with no prior
conversational context to track. Its weakness is exactly what's outside
that scope: instruction-following across multiple edits, keeping a
character or object consistent across a series of generated images, or
reasoning about the request the way an LLM reasons about a text prompt.
The text conditioning is an input to the denoising process, not
something the model is actively reasoning over.

Autoregressive generation's strength is the inverse: because it's the
same substrate as the LLM handling the conversation, it inherits that
model's instruction-following and in-context reasoning, which is why it
tends to be better at multi-turn edits ("now make the background darker
and remove the second person") and at generating an image that's
correct relative to something discussed earlier in the same chat. It
has historically lagged pure diffusion models on raw output fidelity
and detail, though that gap has been closing as labs invest in native
multimodal generation specifically.

The line between the two is blurring on purpose. Several labs now
describe their flagship image generation as coming from the same model
that handles text and vision input, with the older "call out to a
diffusion pipeline" pattern increasingly reserved for image-only
products rather than conversational assistants. Checking which pattern
a given feature actually uses — a genuinely unified model, versus an
LLM front-end that routes a generation request to a separate diffusion
model behind the scenes — explains a lot of the difference between
products that handle "edit this image based on our conversation" well
and ones that treat every generation request as a fresh, contextless
prompt.

## Evaluating image models

Recognition has real benchmarks with graded answers — VQA and DocVQA
for question-answering over natural images and documents, MMMU for
college-level multimodal reasoning — and they carry the same caveats
the [Reading the Spec Sheet](../reading-model-specs/index.md) chapter
raised for text benchmarks: contamination risk on anything old and
public, and a real gap between benchmark accuracy and a model's actual
reliability on the specific document types a given use case cares
about.

Generation quality mostly doesn't have a graded answer to check
against. There's no ground-truth "correct" image for a text prompt, so
the field leans on the same tool text models lean on for the same
reason: human pairwise preference, converted to an Elo-style rating,
run by independent arenas rather than vendor self-reports. The caveat
carries over unchanged too — preference ranking measures what people
like looking at, which correlates with prompt adherence and aesthetic
quality but isn't the same claim as either one measured directly.

## Provenance and licensing

Training data disputes over image models tend to be sharper than the
equivalent disputes over text, because a generated image can resemble a
specific artist's identifiable style in a way generated prose rarely
mirrors a specific author's voice as legibly — several ongoing lawsuits
are about exactly this. Checking a vendor's stated training-data policy
and opt-out mechanism is worth doing before adopting a model into a
commercial pipeline, separately from checking the license on the
weights themselves, for the same reason the
[Lineage and Licenses](../model-lineage-licenses/index.md) chapter
treated a model's license and its acceptable-use policy as two
documents to read separately rather than one.

Provenance on the output side is a newer, still-consolidating answer:
standards like C2PA content credentials and invisible watermarking
schemes like SynthID embed a signal in generated images that tools can
later check to tell whether an image came from a generative model.
Coverage is inconsistent across vendors and images circulating outside
the platform that generated them frequently lose the metadata entirely,
so treat a watermark's absence as inconclusive, not as proof an image
is unedited or human-made.

## The actual key

Recognition and generation are different capabilities running on
different machinery, and the questions worth asking differ accordingly.
For recognition: how is the image being patched and priced, and is the
task one of the three — counting, precise localization, small detail —
where the model's known weaknesses are likely to bite? For generation:
is this diffusion or an autoregressive model reasoning in the same
context as the conversation, and does the task actually need the
multi-turn consistency only the second one reliably gives? And for
either: is the evaluation behind a vendor's claim a graded benchmark, or
a preference ranking measuring something adjacent to correctness rather
than correctness itself?

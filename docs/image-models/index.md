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

An image starts as pixels, but a multimodal language model normally does
not feed raw pixel values straight into its text transformer. A vision
encoder—often a Vision Transformer (ViT)—divides the image into patches
and turns them into feature vectors. A projector, resampler, or similar
adapter then converts those features into a sequence the language model can
consume alongside text tokens. Architectures differ: some keep a distinct
vision encoder, while more unified designs share more of the processing.

That patching has a direct, billable consequence: a high-resolution
image, or a vendor's tiling scheme that splits a large image into
several sub-images to preserve detail, produces more patches — more
tokens — than a small or low-resolution one. Providers commonly convert
this visual work into image-token counts or another resolution-dependent
billing unit. Whether those units share the text context budget and how
they are priced is API-specific, not a fixed industry rule. An image
dropped into a long conversation can quietly
consume a large share of the context window in a way a wall of text of
comparable "size" wouldn't.

## What recognition is reliably good and bad at

Frontier vision models are strong, unsurprisingly, at the things a
transformer trained on internet-scale image-text pairs would be
expected to be strong at: describing a scene, reading printed or
handwritten text, answering questions about a chart or a screenshot,
classifying what kind of document something is. OCR on clean printed text
is often strong, but accuracy still depends on resolution, layout, script,
typography, and whether the product uses a dedicated OCR stage. "Readable
in a demo" is not the same as reliable transcription for production
documents.

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

These weaknesses come from several sources: resizing and patching can lose
detail, language-model decoding favors plausible answers, and generic
training objectives do not enforce exact counting or coordinates. Patches
are learned representations rather than simply averaged pixels, but they
still do not preserve a lossless copy of the source image.

## Generation: two broad approaches

Where recognition converts an image into tokens for a transformer to
read, generation has two prominent approaches. Real products can combine
them, so these are not mutually exclusive boxes.

**Diffusion models** — including Stable Diffusion and Flux — typically
start from random noise and run a series of steps that
predict and subtract a bit of that noise at each step, guided by a text
encoding (commonly from a CLIP-style model or a separate language
model) fed in via cross-attention at every step. Most production
diffusion models do this denoising in a compressed latent space rather
than on raw pixels — a separate encoder compresses the image down and a
decoder expands the final latent back into pixels — which is most of
why modern diffusion generation is fast enough to be interactive rather
than something to leave running for minutes.

**Autoregressive image models** represent an image as a sequence of tokens
and predict them in an order conditioned on prior tokens and the prompt.
The order need not correspond literally to pixels from left to right.
Some native multimodal systems can generate text and image representations
within one model family, but vendors do not always publish enough
architecture detail to conclude that the exact same transformer produces
both or that no diffusion-style decoder is involved.

## Why the architecture choice matters in practice

Diffusion systems have produced strong image quality and can generate in
parallel within each denoising step, though they require multiple such
steps. Their instruction-following depends heavily on the text encoder,
conditioning design, training data, and any language model placed in front
of the generator. Diffusion itself does not imply weak editing or poor
multi-turn consistency; those are properties to test in the complete
product.

Autoregressive generation can integrate naturally with a token-based
multimodal context, which may help instruction-following and conversational
editing. That advantage is not automatic: the surrounding product may give
a diffusion generator the same conversation through prompt rewriting and
image conditioning, while an autoregressive model can still lose identity
or detail between turns. Architecture alone does not rank the resulting
products.

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
multi-turn consistency the product demonstrates? And for
either: is the evaluation behind a vendor's claim a graded benchmark, or
a preference ranking measuring something adjacent to correctness rather
than correctness itself?

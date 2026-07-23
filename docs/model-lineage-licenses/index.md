---
icon: lucide/scroll
---

# Models: Lineage and Licenses

Two things worth knowing before comparing any model on the spec sheet:
where the current generation of models actually came from, and what
you're legally allowed to do with the one you're looking at. Neither is
covered by a benchmark table. They're unrelated questions, so this
chapter covers them as two separate halves rather than one continuous
story.

## A short history

This picks up where [Theory: Machine Learning Before
LLMs](../machine-learning-before-llms/index.md) leaves off. Before
transformers, sequence models were recurrent: an RNN or LSTM read
a sentence one token at a time, carrying a running summary forward step
by step. That worked, within limits — recurrent networks had difficulty
learning long-range dependencies, in part because gradients could vanish
or explode during training, and because each step
depended on the one before it, training couldn't be parallelized across
a sequence the way it could across a batch. Both limits shaped what came
next more than any single new idea did.

The 2017 paper "Attention Is All You Need" removed recurrence from its
sequence model. Self-attention lets positions exchange information
directly, while masks control which positions are visible: an encoder can
attend in both directions, while an autoregressive decoder cannot attend to
future tokens. Training can compute all positions in a sequence in parallel,
although generation still produces tokens sequentially. That parallel
training and the architecture's scaling behavior helped make much larger
models practical.

GPT-1 (2018) was an influential demonstration of what that architecture
enabled when combined with a specific training recipe:
unsupervised pretraining on raw text, followed by supervised fine-tuning
on a specific task, showed that the language understanding picked up in
pretraining transferred to tasks the model was never directly trained
on. BERT, released the same year, took the transformer down a different
branch — bidirectional and encoder-only, trained by masking words and
predicting them from both sides, better suited to classification and
understanding tasks than to generating fluent text.

GPT-2 (2019) was a straightforward scale-up of GPT-1, notable at the
time less for the architecture than for the release itself: OpenAI
initially withheld the full model, citing misuse concerns, and released
progressively larger versions over several months rather than all at
once. That staged rollout was the first time "how a model gets released"
became its own decision, separate from "how a model gets built" — a
distinction that matters a great deal in the licensing half of this
chapter.

GPT-3 (2020) scaled further and demonstrated few-shot in-context
learning — the model performing a new task from a handful of examples in
the prompt, no fine-tuning required. It also marked a shift in
distribution: no weights were released at all, only API access. Where
GPT-1 and GPT-2 had been published for anyone to inspect and run, GPT-3
was the point where a frontier model became something you called rather
than something you could download.

The step from GPT-3 to something people actually wanted to talk to was
mostly a training-recipe change, not an architecture change:
reinforcement learning from human feedback (RLHF), which fine-tunes a
model against human ratings of its responses so it learns to follow
instructions and produce answers people rate as helpful, rather than
just predicting the statistically likely next token. ChatGPT (November
2022) was that recipe applied to a GPT-3.5-class model with a chat
interface in front of it — the moment broad public adoption happened,
built on techniques that had existed for a while rather than a new
architectural leap.

From there the field split into two tracks that still define it: closed,
API-only frontier labs (OpenAI, Anthropic, Google), and an open-weight
counter-track that gathered real momentum once Meta started publishing
Llama's weights in 2023, followed by Mistral, DeepSeek, and Qwen. That
split — who publishes weights and under what terms — is exactly where
the second half of this chapter picks up.

## Licenses

"Open weight" describes distribution — the weights are downloadable —
and says nothing on its own about what you're allowed to do with them.
That's governed by a separate document, and it's worth checking every
time rather than assuming the word "open" in a release's name settles
the question.

**Genuinely permissive releases** exist: some models ship under a
standard permissive license like Apache 2.0 or MIT, which generally permits
commercial use, modification, and redistribution subject to conditions such
as preserving notices. Apache 2.0 also includes express patent terms.
Applying a software license to weights can still leave separate questions
about training data, accompanying code, trademarks, and applicable law, so
the short license text is not the whole diligence exercise.

**Custom "open-weight" licenses** are more common at the frontier end of
open releases, and they routinely carry restrictions a standard
open-source license wouldn't: a usage-scale threshold above which a
commercial user needs a separate agreement directly with the vendor
rather than relying on the published license (Llama's license works this
way), and an acceptable-use policy layered on top of the license itself
that names specific prohibited uses. Some licenses or policies also restrict
using the model or its outputs to train a competing model. That clause is
worth checking specifically, but it is not universal across open-weight
releases and its wording varies.

**Closed, API-only models** don't have a software license at all in the
usual sense, since there are no weights to license — what governs them
is a terms of service and acceptable-use policy attached to the API
itself. The clauses worth checking there are different in kind:
whether outputs may be used to train another model (commonly forbidden,
for the same anti-distillation reason as above), who owns the output
(typically the customer, per the vendor's terms), and what happens to
submitted input — whether it's used for further training by default, and
whether there's an opt-out.

The one habit that covers all three cases: don't infer permission from
the word "open," a family name, or a license's general reputation.
Check whether the specific document is a standard open-source license or
a custom one, read the acceptable-use policy as a separate document from
the license itself, and specifically look for a scale threshold and a
competing-model-training clause, since those two are where "open enough
to download" and "open enough to build a business on" most often stop
being the same thing.

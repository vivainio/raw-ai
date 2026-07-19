---
icon: lucide/gauge
---

# Models: Reading the Spec Sheet

Every model release comes with the same two artifacts: a spec sheet
(parameters, context window, modalities) and a bar chart of benchmark
scores where the new model's bar is always tallest. Neither is lying,
exactly, but neither is built to be compared apples-to-apples across
vendors either. This chapter is a key for reading that page — what each
number actually measures, where it's soft, and what a vendor's own
comparison table is likely to be hiding. The numbers themselves will be
out of date within months; the questions to ask about them won't be.

## Parameters mean less than they used to

Parameter count used to be the headline: GPT-3's 175B was the whole
pitch. Frontier labs mostly stopped publishing it — OpenAI, Anthropic, and
Google don't state a parameter count for their current flagship models at
all. That's not an oversight. Once it became clear that training data,
post-training, and architecture choices move a model's quality more than
raw scale does, a bare parameter count stopped being a useful marketing
number and started being a number a competitor could use to estimate
your training cost. Withholding it costs the vendor nothing, since an API
customer never touches the weights and has no use for the figure anyway.

Open-weight releases (Llama, Mistral, Qwen, DeepSeek) still publish exact
counts, and for a concrete reason: you're downloading the weights, and
the count tells you what GPU memory you need to run them. The number
serves the reader there in a way it doesn't for a hosted API.

Where a count does get published,
[mixture-of-experts](../transformer-architecture/index.md#mixture-of-experts-not-every-parameter-runs-for-every-token)
architectures split it into two figures that get conflated constantly:
total parameters and active parameters per token. A model can list a large
total count while only routing each token through a fraction of it — active
parameter count tracks much closer to inference cost and latency than total
does. Seeing one big number in a headline without total-vs-active broken
out is a sign to go find the breakdown before treating it as comparable to
a dense model's parameter count.

## Context window: advertised max, not effective use

"1M token context window" describes the longest input the model will
accept, not how reliably it uses everything in that input. The two are
tested separately, and they diverge in predictable ways.

The standard first test is needle-in-a-haystack: bury one fact in a long
document and ask the model to retrieve it. Most current frontier models
pass this cleanly at full length. It's a floor, not a ceiling — real
tasks rarely need one isolated fact pulled from noise. Harder variants
move closer to what actually happens in practice: multiple needles
scattered through the same context, needles that require connecting two
separated facts rather than quoting one verbatim, or "lost in the
middle" effects where recall measurably drops for content placed in the
middle of a long input versus the start or end. A model can ace the
simple version of this test and still degrade on the harder ones at the
same context length.

The practical habit: treat the advertised window as an upper bound on
what you can send, and treat effective long-context behavior — recall in
the middle, reasoning across scattered facts — as a separate claim that
needs its own evidence, not something the max-length number implies.

## The benchmark alphabet soup

Vendor comparison tables lean on a recurring set of named benchmarks.
Knowing what each one actually tests — and where it's known to be weak —
is most of what's needed to read the table critically instead of just
reading the bar heights.

| Benchmark | What it tests | Where it's weak |
|---|---|---|
| MMLU / MMLU-Pro | Broad academic knowledge across subjects | Old and public enough that contamination is a live concern; frontier models cluster near the ceiling, so small gaps aren't meaningful |
| GPQA (Diamond) | Graduate-level science questions written to resist lookup | Small question set — a handful of leaked or ambiguous questions move the score noticeably |
| HumanEval / MBPP | Short, self-contained coding problems | Largely saturated at frontier level; doesn't resemble real multi-file coding work |
| SWE-bench (Verified vs. full) | Resolving real GitHub issues, usually agentically | Verified is a human-filtered subset chosen for solvability, so it isn't directly comparable to the full set; solve rate also depends heavily on how many tool calls and retries the scaffold was allowed, which vendors don't always report alongside the score |
| AIME / MATH | Competition and olympiad-level math | New AIME problems are published yearly, which doubles as a natural contamination check — compare a model's score on problems from before versus after its training cutoff |
| LMArena (Chatbot Arena) | Human pairwise preference, converted to an Elo-style rating | Measures what people rate highest, not correctness — reliably gameable by longer, more confidently-worded answers regardless of whether they're right |

None of these are wrong to report. The point of the table is to know, for
each row, whether it's testing the skill you actually care about or a
proxy for it that happens to correlate loosely.

## Reading a vendor's own comparison table

A few recurring patterns are worth checking for specifically before
trusting a vendor-published bar chart:

- **Mismatched scaffolding.** The new model's score often comes with best-
  of-n sampling, tool use, or extended reasoning turned on, compared
  against a competitor run with none of that enabled. The gap being
  measured is partly the scaffold, not the model.
- **Baseline picking.** The comparison target is sometimes a competitor's
  smaller or cheaper tier, or a snapshot that's since been superseded,
  rather than that competitor's current flagship.
- **Selective reporting.** The table shows the handful of benchmarks the
  new model wins on. A benchmark it doesn't lead on simply doesn't
  appear, and its absence is easy to not notice.
- **Contamination tells.** A suspiciously high score on an old, public,
  widely-cited benchmark paired with a much less impressive score on a
  newer or private benchmark testing the same skill is the classic
  signature of test data having leaked into training data somewhere
  upstream.

None of this means the number is fabricated. It means a single bar in a
vendor's own chart is the least independent piece of evidence available
about that model, not the most.

## Pricing as a signal, not a cost

Per-token pricing is one of the few figures a vendor can't spin, since
it's contractual rather than a chosen benchmark. Where it's useful isn't
the absolute rate but the pattern across a vendor's own lineup: relative
price differences between that vendor's tiers track relative compute
budget allocated to each one reasonably well, and a sibling model priced
well below the flagship is reliably the smaller, distilled model in the
family rather than a repackaging of the same weights.

One thing to watch for when a model does its thinking as visible or
hidden reasoning tokens: that reasoning text is generated and billed the
same as any other output, even when the interface only shows a summary
or hides it entirely. A model that looks cheap per token can still cost
more for a given task than one with a higher quoted rate, if it reasons
at length before answering. Comparing cost per completed task, not just
the headline per-token figure, is the only version of this comparison
that means anything.

## Independent checks over vendor self-reports

Vendor-published numbers are run on the vendor's own scaffold, prompting,
and hardware, which is exactly the setup a comparison table needs to
control for and can't be trusted to. Third-party aggregators that run
every model through the same harness and the same prompts are worth
weighting more heavily than any single vendor's own table, precisely
because the scaffold is held constant across the row instead of tuned
per model.

One independent framing worth knowing about specifically because it
measures something the benchmarks above don't: rather than a pass/fail
score on fixed questions, it tracks the length of task — measured in how
long a skilled human would take to do it — that a model can complete
autonomously at a given reliability threshold. The trend of that number
over successive model releases says more about real usefulness on
open-ended work than any single accuracy percentage does.

## Naming and versioning

Family names encode size even when the spec sheet doesn't: a "mini",
"nano", "flash", or similarly-suffixed model is the smaller, distilled
sibling of that family's flagship, not a separate variant of it, and
comparing its scores against another vendor's flagship without noting
that is comparing across a tier gap.

Version pinning matters for the same reason a floating dependency does
in any other software: a dated snapshot name stays fixed, but an alias
like "latest" or the bare family name can move to a different underlying
model without warning. Behavior can change under an application that
never changed its own code, simply because the alias it called moved.
Anything where reproducibility matters is worth pinning to a dated
snapshot rather than an alias, for the same reason a lockfile beats a
bare version range.

## The actual key

Faced with a spec sheet or a benchmark table, the questions that matter
are the same regardless of which model is on it: What exactly does this
number measure, and what does it not? Was it run on the vendor's own
scaffold or an independent one holding the scaffold constant? Is this a
flagship or a smaller sibling in the family? And is the source a dated
snapshot or a moving alias? The specific bars will be stale by the next
release. Those questions won't be.

---
icon: lucide/server
---

# Hosting: AWS Bedrock

The [AWS AgentCore](../aws-agentcore/index.md) chapter took apart a dozen
agent-shaped services and left one thing unexplained: every one of them
still has to call a model somewhere, and that call goes to Bedrock.
AgentCore's Harness picks a model provider from a config field —
"Bedrock, OpenAI, Gemini, or anything LiteLLM-compatible" — and Bedrock is
the first of those options, AWS's own answer to "how do I call a
foundation model without going to that provider's API directly." It's the
layer underneath the agent stack, not another entry in it, which is why
it gets its own chapter instead of a slot in that one.

## What it actually is

Bedrock is a managed API surface over foundation models from several
providers — Anthropic, Meta, Mistral, Cohere, AI21, Stability AI, and
Amazon's own Nova and Titan lines — reached through one API, one IAM
boundary, and one bill, instead of a separate account and SDK per
provider. On top of that base sit a handful of adjacent services: Guardrails
(content and PII filtering), Knowledge Bases (managed RAG), Agents (a
config-based agent product that predates AgentCore), Flows (a visual
chain builder), Custom Model Import (bring your own weights), and two
extra ways to buy compute beyond pay-per-token. Each is its own API and
its own section below; none of them requires the others.

## The Converse API: one shape across providers

Bedrock actually ships two ways to call a model. `InvokeModel` is the raw
path — the request and response bodies are whatever the underlying
provider defines, so an Anthropic model still expects Anthropic's message
format and a Titan model expects Titan's, just wrapped in an AWS envelope
and billed through AWS instead of called directly. `Converse` is the
newer, unified path: one request shape and one response shape that work
against any Bedrock model that supports it, so switching providers is a
change to `modelId`, not a rewrite of the request body.

```python
import boto3

client = boto3.client("bedrock-runtime", region_name="us-east-1")

response = client.converse(
    modelId="us.anthropic.claude-sonnet-4-5-20250929-v1:0",
    messages=[{"role": "user", "content": [{"text": "Summarize this in one sentence."}]}],
    system=[{"text": "Be concise."}],
    inferenceConfig={"maxTokens": 512, "temperature": 0.3},
)
reply = response["output"]["message"]["content"][0]["text"]
```

Swap that `modelId` for a Llama or Nova model and every other line stays
the same — `messages`, `system`, `inferenceConfig`, and the
`output.message.content` shape on the way back are all provider-agnostic.
That's a real abstraction, the same job LiteLLM or OpenRouter do outside
AWS, just run by AWS itself with AWS billing and AWS IAM controlling who
can call it, rather than a separate service sitting in front of
everyone's API keys.

Tool calling rides the same unification. A `toolConfig` block —
`tools`, each with a `name`, `description`, and JSON Schema
`inputSchema` — produces the same `tool_use`/`tool_result` loop described
in the [Agent Dialogue](../agent-dialogue/index.md) chapter, regardless of
which model is behind `modelId`:

```python
response = client.converse(
    modelId="us.anthropic.claude-sonnet-4-5-20250929-v1:0",
    messages=messages,
    toolConfig={"tools": [{"toolSpec": {
        "name": "get_weather",
        "description": "Get current weather for a city",
        "inputSchema": {"json": {"type": "object", "properties": {"city": {"type": "string"}}, "required": ["city"]}},
    }}]},
)
if response["stopReason"] == "tool_use":
    tool_block = next(b["toolUse"] for b in response["output"]["message"]["content"] if "toolUse" in b)
    # run tool_block["input"], send a toolResult message back on the next converse() call
```

Model-specific tool-calling quirks — how Anthropic wants tool results
formatted versus how Llama does — get absorbed into `Converse` before
they ever reach your code. Call `InvokeModel` instead and that absorption
doesn't happen; you're back to whatever format the model natively
expects.

## The `us.` prefix isn't decorative

That `modelId` above isn't a raw model name — the `us.` prefix is a
**cross-region inference profile**, and for a growing list of models
(most current Claude versions among them) it's not optional. Call the
bare model ID directly and Bedrock returns a validation error saying
on-demand throughput isn't supported for that model; it has to be
addressed through an inference profile instead. The profile routes the
call across whichever AWS regions in that geography (`us.`, `eu.`,
`apac.`) have capacity, which is the actual point of it — smoothing over
regional capacity limits during traffic spikes — but the visible effect
for a caller who doesn't know that is a model ID that looks like it
should work and doesn't, until a region prefix gets tacked onto the
front. Worth knowing before debugging a "model not found"-shaped error
that's actually a missing profile prefix.

## Model access: mostly automatic now, with two exceptions

Bedrock used to require enabling every single model by hand in the
console before a single call to it would succeed — a manual
"model access" request per model, sometimes with an approval delay, that
tripped up first-time callers constantly. That requirement is mostly
gone: serverless models are enabled by default in every region they're
offered, and the model-access screen and its underlying API call have
been retired for most providers.

Two exceptions survive that simplification, and both are worth knowing
before assuming "any model, any account, first call succeeds." Anthropic
models require a one-time use-case acceptance — a short form submitted
once per account, either in the console playground or through the
`PutUseCaseForModelAccess` API — before the first call to any Anthropic
model on Bedrock goes through; every call after that first acceptance
works normally. And a subset of models are distributed through AWS
Marketplace rather than directly, which means a subscription step
(`CreateFoundationModelAgreement` or the console equivalent) before
anyone in the account can call them, a separate gate from the Anthropic
form and specific to whichever models happen to be Marketplace-listed.

## Guardrails: filtering decoupled from the model call

Guardrails governs a different boundary than
[Policy](../aws-agentcore/index.md#policy) does. Policy gates which
*tools* an agent is allowed to call through an AgentCore Gateway.
Guardrails inspects *text* — a prompt going in, a completion coming out —
against content filters (hate, insults, sexual content, violence,
misconduct, and prompt-injection detection), denied topics, a
custom-word blocklist, PII detectors, and a contextual-grounding check
that flags a RAG answer that isn't actually supported by the retrieved
text. Neither one implies the other: a Guardrail can run in a pipeline
with no tools at all, and a tool call gated by Policy carries no text
inspection of its own.

A guardrail can attach directly to a `Converse` call:

```python
response = client.converse(
    modelId=model_id,
    messages=messages,
    guardrailConfig={"guardrailIdentifier": guardrail_id, "guardrailVersion": "1", "trace": "enabled"},
)
```

or run standalone through `ApplyGuardrail`, which checks a block of text
against a guardrail without invoking any model at all:

```python
result = client.apply_guardrail(
    guardrailIdentifier=guardrail_id,
    guardrailVersion="1",
    source="INPUT",
    content=[{"text": {"text": user_message}}],
)
if result["action"] == "GUARDRAIL_INTERVENED":
    blocked_or_masked_text = result["outputs"][0]["text"]
```

That standalone form is the more flexible of the two, because it isn't
tied to a Bedrock model invocation at all — the same guardrail can screen
a prompt before a RAG retrieval even runs, check output from a model
called outside Bedrock entirely, or sit in front of a Custom Model Import
deployment the same way it sits in front of Anthropic's models. The
inline `guardrailConfig` on `Converse` is the convenience path when the
model call and the check are always going to happen together; `ApplyGuardrail`
is what a pipeline reaches for when they don't.

## Knowledge Bases: managed RAG, same shape as Memory's split

Point a knowledge base at an S3 prefix, pick or let AWS provision a
vector store, and Bedrock runs the ingestion pipeline: chunk each
document, embed the chunks, write the vectors, and keep the index in
sync as the source data changes. The vector store itself defaults to an
auto-provisioned OpenSearch Serverless collection, but Aurora, Neptune,
Pinecone, Redis, and the newer S3 Vectors (embeddings queried directly
out of S3, no separate database to run) all work as a customer-managed
alternative.

Retrieval is one API call, and comes in two depths:

```python
agent_client = boto3.client("bedrock-agent-runtime")

# just the matching chunks
chunks = agent_client.retrieve(
    knowledgeBaseId=kb_id,
    retrievalQuery={"text": "What's the refund policy?"},
)

# matching chunks *and* a generated answer built from them
answer = agent_client.retrieve_and_generate(
    input={"text": "What's the refund policy?"},
    retrieveAndGenerateConfiguration={
        "type": "KNOWLEDGE_BASE",
        "knowledgeBaseConfiguration": {"knowledgeBaseId": kb_id, "modelArn": model_arn},
    },
)
```

`retrieve` hands back the raw chunks for a caller that wants to do its
own prompt assembly; `retrieve_and_generate` does the retrieval and the
generation step in one call, citations included.

The shape here is the same one [AgentCore Memory](../aws-agentcore/index.md#memory)
uses for long-term memory: raw data goes in, a background process AWS
runs turns it into the thing actually queried, and the query itself is a
similarity search rather than a key lookup. Memory's background step is
an extraction job over conversation events; a knowledge base's is a
chunk-and-embed job over documents. Same pattern — write path is the
easy part to build yourself, the embedding pipeline and the index it
feeds is the part a knowledge base actually saves you from standing up.

## Agents for Amazon Bedrock: the product AgentCore didn't replace

This is the piece most likely to confuse a reader arriving from the
AgentCore chapter, because the name overlaps and the underlying AWS
service does not. "Agents for Amazon Bedrock" — the classic product,
sometimes called Bedrock Agents to avoid the ambiguity — predates
AgentCore by about two years and is still a separate, still-supported
offering, not a deprecated predecessor folded into it.

Classic Agents is fully configuration-driven: pick a model, write
instructions, define **action groups** (each one a Lambda function plus
an OpenAPI or function schema describing what it does — the same
"turn an API into a callable tool" job [Gateway](../aws-agentcore/index.md#gateway)
does for AgentCore), optionally attach a knowledge base, and Bedrock
handles orchestration, prompt assembly, and session state without any of
that being code you write. It's a narrower, older bundle than AgentCore's
combination of Runtime, Gateway, Memory, and Harness — one Lambda-backed
tool shape instead of Gateway's OpenAPI/Smithy/Lambda targets, one fixed
orchestration strategy instead of a framework choice — but it's also
simpler to stand up for exactly that reason, and AWS documents it as a
live alternative rather than a legacy path to migrate off. The practical
split: classic Agents fits a small, Lambda-shaped tool set where the
built-in orchestration is good enough; AgentCore fits anything wanting
framework choice, non-Lambda tool sources, or the rest of AgentCore's
pieces (Browser, Code Interpreter, Identity) that classic Agents was
never built to reach.

## Flows: the no-code answer to a graph

[Strands' `Graph`](../strands-agents/index.md#multi-agent-orchestration-is-a-first-class-object)
builder wires prompt/agent/tool nodes together in Python, with explicit
edges between them. Flows does the same job as a drag-and-drop canvas in
the Bedrock console instead of a Python object: nodes for prompts (pulled
from Bedrock's versioned prompt catalog), models, knowledge bases,
guardrails, classic Agents, Lambda functions, and conditional-logic
branches, connected by dragging an output into the next node's input.
The result is testable and traceable from the same console, input and
output logged at every node.

It's the closest thing Bedrock has to Harness's "config instead of code"
trade from the AgentCore chapter, but aimed at a different shape of
problem: Harness configures one agent's loop — model, tools, memory —
and lets the model decide what happens next. Flows configures a fixed
pipeline where a human decided the steps and their order ahead of time,
closer to a workflow engine than an agent loop. Reach for Harness when
the model should be the one deciding what to call next; reach for Flows
when the sequence is already known and what's being configured is which
prompt, model, and knowledge base sit at each fixed step.

## Three ways to pay for the same call

On-demand, per-token billing is the default and the one every example
above uses without any setup. Two more purchasing shapes sit next to it,
each solving a different problem than raw cost:

**Provisioned Throughput** reserves dedicated capacity — measured in
Model Units, each one a fixed amount of input/output tokens per minute —
for a commitment term (none, one month, or six months) instead of
billing per call. The reason to reach for it is guaranteed throughput,
not spend: on-demand calls can be rate-limited under load, provisioned
capacity can't, which matters for a workload that needs a steady,
predictable ceiling rather than best-effort. Some models require
provisioned throughput before they can be called at all — Custom Model
Import deployments among them — so it's not purely optional for every
model either.

**Batch inference** goes the other direction — no throughput guarantee,
higher latency, in exchange for cost. `CreateModelInvocationJob` takes a
pointer to a batch of prompts sitting in S3, runs them asynchronously
against a model with no live turnaround, and writes results back to S3
when the job finishes. It fits exactly the workloads Code Interpreter's
"delegate the data work to code" argument in the AgentCore chapter
describes — bulk, non-interactive processing — just applied to model
calls at volume instead of a script, and it's the shape to reach for over
a live `Converse` loop when the caller doesn't need any single answer
back in real time.

## Custom Model Import: bring your own weights, skip the GPUs

Fine-tuning inside Bedrock — continued pre-training or supervised
fine-tuning on a base model AWS already hosts — is one path to a
customized model. Custom Model Import is a different one: the model is
already trained somewhere else entirely, and what Bedrock takes is the
finished weights. Point it at Hugging Face `safetensors` files sitting in
S3 or SageMaker — a fine-tuned or domain-adapted open-weight model — and
Bedrock imports it, serves it through the same `Converse`/`InvokeModel`
surface as any first-party model, and can wire it into Knowledge Bases,
Guardrails, or classic Agents the same way. Supported architectures cover
a specific, growing list rather than "anything" — Llama, Mixtral, Qwen,
DeepSeek-R1 distills, and OpenAI's open-weight GPT-OSS models among them —
so it's a serving layer for a known set of open architectures, not a
generic "upload any checkpoint" import.

The trade it makes explicit: no GPU fleet to provision, patch, or scale
for a model you already own the weights for — that infrastructure
problem moves to AWS the same way Runtime moves container hosting off
your own compute in the AgentCore chapter. What doesn't move is the
training itself; getting to those weights in the first place is still
entirely your own pipeline, on whatever hardware trained them.

## What Bedrock actually is, once the layers are separated

Set aside marketing and the honest description is a model-hosting API
with a handful of genuinely useful services layered on top of it, not an
agent framework and not a competitor to AgentCore, Strands, or Pydantic
AI — all three of those call into Bedrock rather than replace it.
`Converse` is the one piece of real, provider-spanning abstraction:
switching model providers becomes a config value instead of a rewritten
request body, the same normalization LiteLLM does outside AWS, run here
under AWS's own IAM and billing instead of a separate broker. Guardrails
and Knowledge Bases are managed versions of things a team could build
themselves — content filtering, chunk-and-embed RAG — with the same gap
plain DIY infrastructure always has against a managed offering: the
pipeline behind the API, not the API surface itself, is the part that's
actually hard to stand up and keep running.

None of it makes a model reason any better, same closing point the
AgentCore chapter lands on — the intelligence is whatever's behind
`modelId`, and Bedrock's job stops at getting a request to that model and
a response back, reliably, inside one AWS account.

---
icon: lucide/cloud
---

# Hosting: AWS AgentCore

Amazon Bedrock AgentCore is marketed as "an agentic platform for building,
deploying, and operating highly effective agents securely at scale." That
sentence is true and says almost nothing. The useful question isn't what
AWS calls it — it's what you actually get for the bill, and how much of it
is new versus an existing AWS primitive wearing an agent-shaped label.

## What ships under the name

AgentCore isn't one service. It's a bundle of a dozen separately billed
pieces, and the list has grown since launch:

| Piece | What AWS says it does |
|---|---|
| Runtime | Serverless, session-isolated environment for running agent code |
| Harness | A managed agent loop — model + tools + prompt in, orchestration handled for you |
| Gateway | Turns APIs and Lambda functions into MCP tools without rewriting them |
| Memory | Short-term (session) and long-term (cross-session) storage for agent context |
| Identity | Token/credential broker between agents and existing identity providers |
| Browser | Managed headless-browser sandbox for web-using agents |
| Code Interpreter | Managed sandbox for agent-executed code |
| Observability | OpenTelemetry traces into CloudWatch |
| Policy | Cedar-based rules gating which tools an agent can call |
| Payments | Wallet integration for agent-initiated microtransactions (x402) |
| Evaluations / Optimization | Automated scoring and A/B tuning of agent traces |
| Registry | A catalog for internally publishing agents, tools, and MCP servers |

Each of these is its own line item, its own pricing unit, its own docs
page. "Platform" is the right word in the sense that it's a lot of
surface area — not in the sense that it's one coherent thing.

## The load-bearing pieces, taken apart

### Runtime

Runtime is the core: a serverless microVM per session, metered by actual
compute time down to the second rather than a fixed instance you keep
running. That's the same execution model Lambda has used for years
(Firecracker microVMs), retuned for a workload Lambda handles badly — an
agent turn that might run for minutes, hold state across tool calls, and
not fit a 15-minute hard ceiling. If you've ever tried to wedge a
LangGraph loop into Lambda and hit that ceiling, Runtime is the answer to
that specific problem, not a new idea.

You still bring your own container. Build a Docker image (ARM64 only —
build it on x86 and dependencies fail silently at runtime), expose
`POST /invocations` and `GET /ping` on port 8080, push it to ECR. Runtime
pulls that image into the microVM per session. This isn't an escape hatch
for "custom environments" — it's the default shape of every agent you
deploy to it. What AWS is selling isn't a way to avoid packaging your own
container; it's what happens to that container once it's running:
per-session isolation, fast cold starts, and a bill measured in vCPU- and
GB-seconds instead of a fixed instance.

The base image itself is nothing special either — `python:3.13-slim` or
equivalent, no AgentCore-branded layer to pull. The only thing that ties
the container to AgentCore at all is one Python package, `bedrock-agentcore`,
and inside it a single decorator: `BedrockAgentCoreApp`. It wraps whatever
function you point it at so that a plain Python call turns into the
`/invocations`/`/ping` contract Runtime expects — request parsing,
streaming, session headers, all handled by that one wrapper.

Because the wrapper only cares about "function in, response out," it
doesn't know or care what's inside the function. Any agent framework
plugs in the same way — here it's wrapping an agent built with a
structured-output framework:

```python
from bedrock_agentcore import BedrockAgentCoreApp
from some_agent_framework import Agent

app = BedrockAgentCoreApp()
agent = Agent(model, instructions="Be concise.")

@app.entrypoint
def invoke(payload: dict, context=None) -> str:
    result = agent.run_sync(payload["prompt"])
    return result.output

if __name__ == "__main__":
    app.run()
```

Swap `some_agent_framework` for LangGraph, CrewAI, Strands, or nothing at
all — a bare function calling the model directly works too. AWS didn't
build a connector for any specific agent framework, and none of them
built one for AWS; both sides just happen to agree on "a Python function
that returns a string or a stream." That's the entire integration
surface. Everything framework-specific — tool definitions, retries,
structured output validation — stays exactly as it was outside AgentCore;
Runtime never sees it.

### Browser

Browser is metered exactly the same way as Runtime — same unit, same
rate. That's the tell — it isn't a bespoke product, it's Runtime with a
different container image and a product name: a managed headless browser
(Playwright- and BrowserUse-compatible) with a live view for watching a
session and a recording for replaying one later. One genuinely specific
feature sits on top: it can cryptographically sign outbound HTTP requests
so a site sees a legitimate, attributable client instead of generic
headless-Chrome traffic, which cuts down on CAPTCHA challenges. Everything
else about it — the sandbox, the isolation, the metering — is Runtime.
It competes directly with Browserbase, E2B, or Cloudflare's Sandbox SDK,
and the pitch is the managed sandbox plus the AWS IAM boundary around it,
not novel execution technology.

### Code Interpreter

Code Interpreter is the same story again: another container image on the
same Runtime substrate, this one pre-loaded with Python, JavaScript, and
TypeScript kernels instead of a browser. Same metering, same per-session
isolation. If Runtime, Browser, and Code Interpreter feel like
three products, that's the packaging talking — underneath they're one
execution primitive wearing three skins.

### Gateway

Gateway is the one piece here that saves real work. Point it at an
OpenAPI spec or a Lambda function and it generates an MCP-compatible tool
endpoint — with inbound and outbound auth wired in — without you writing a
tool wrapper by hand. It also does semantic and keyword search over the
tools it exposes, so an agent with access to hundreds of tools doesn't
need all of their descriptions stuffed into context just to pick the
right one. If you were going to hand-roll "expose fifteen internal APIs
to an agent as tools," Gateway is a genuine shortcut, not a rebrand.

### Identity

Identity doesn't replace Cognito, Okta, or Auth0 — your users still
authenticate through whichever of those you already run. What it adds is
a **workload identity**: a new identity primitive, distinct from any IAM
role or Cognito user, that represents the agent itself and gets its own
ARN. Inbound Auth checks who's allowed to call the agent (IAM or an OAuth
JWT); Outbound Auth is what the agent presents to reach *other* things on
the caller's behalf — via a token vault that supports user-delegated
access (OAuth authorization-code grant), service-to-service calls (client
credentials grant), and on-behalf-of token exchange, so an agent can act
for an authenticated user against a downstream API without re-prompting
for consent. That last piece is a real, if narrow, addition — most clouds
already have a "workload identity" concept for service-to-service calls
(Kubernetes and GCP both ship one), and AgentCore's version is that
pattern extended to a caller that happens to be a model, not a wholly new
idea.

### Memory

Memory has two tiers, and they work differently. Short-term memory is
just a session-scoped event log: every message gets appended, nothing is
interpreted. Long-term memory is generated from that log after the fact —
a background job runs one of four built-in extraction strategies
(semantic facts, user preference, summary, or episodic behavior-over-time)
and writes the result as a separate, queryable record. So the raw
conversation and the thing you actually search later aren't the same
data — the log is input, the extraction step is what produces the
searchable memory. That's a managed table plus an LLM-powered extraction
step, not a new memory architecture. Compare it to the
[agent memory](../agent-memory/index.md) a coding agent keeps in a flat
file on disk: same idea — write notes now, read them back later when
relevant — except here "decide what's worth remembering" is a separate,
asynchronous step AWS runs for you instead of something the agent does
inline while writing the file.

None of that is wired to the model call, though. Memory is a
store-and-retrieve API — writing an event and reading memories back are
two explicit calls your code makes, at opposite ends of the loop:

```python
# after a turn: append to the raw event log
client.create_event(
    memory_id=memory_id,
    actor_id=actor_id,
    session_id=session_id,
    messages=[(user_text, "USER"), (reply_text, "ASSISTANT")],
)

# before the next turn: pull back whatever got extracted
memories = client.retrieve_memories(
    memory_id=memory_id,
    namespace=f"/summaries/{actor_id}/{session_id}/",
    query=next_user_text,
)
```

`retrieve_memories` returns text — nothing splices it into the prompt for
you. That's still your code, same as it would be with any vector store:
concatenate it onto the next message or the system prompt yourself before
the model call, e.g. `messages[-1]["content"] = f"Context:\n{memories}\n\n{original_text}"`.
Some agent frameworks expose "before model call" / "after model call"
hooks so this read-then-write pair can live at the edges of the loop
instead of threaded through every call site — but the hook is just a
place to put the same manual glue, not automation that replaces it.

Which raises the obvious question: how is any of this different from
just writing events to a DynamoDB table yourself? For short-term memory,
it mostly isn't — that's a raw log keyed by actor and session, a shape
you could stand up yourself in an afternoon, probably for less money.
Long-term memory is where the real gap sits, and it's two things, not
one. First, `retrieve_memories` isn't a key lookup — the `query` argument
gets embedded and matched by similarity against the stored records, which
means there's a managed vector index behind it. Plain DynamoDB has no
such thing; getting this yourself means standing up OpenSearch, pgvector,
or a dedicated vector store and keeping its embeddings in sync with every
write. Second, the raw event log isn't what gets searched at all — a
background job reads it and produces the distilled records retrieval
actually queries, which means an LLM call, a decision about what's worth
keeping, and presumably some de-duplication over time, all running on
AWS's schedule instead of yours. Storing events yourself gets you the
easy half of this — the write path. The half you'd actually have to build
is the embedding pipeline and the summarization job sitting between the
raw log and the thing you query — and that gap isn't specific to AWS;
it's the same gap any RAG-over-conversation-history setup has to close,
however it's built.

Isolation between users is `actor_id`-scoped, and the granularity is
whatever namespace template you write, not something fixed. `/actor/{actorId}/`
persists across all of that user's sessions — where semantic facts and
preferences live, since "prefers window seats" shouldn't reset every
conversation — while `/actor/{actorId}/session/{sessionId}/` scopes a
summary to one conversation. Nothing stops a coarser namespace either;
`actor_id` is just a string your code supplies, not a fixed identity
primitive, so a shared `/team/{teamId}/` namespace works exactly the same
way. Two things worth knowing about that string, though: AWS never
derives it from anything — it isn't wired to Identity's workload
identity or to Cognito, so the isolation guarantee is only as good as
your own code's discipline about which value it passes as `actor_id`. And
IAM can enforce the boundary rather than just assume it — actor, session,
and namespace can all be used as context keys in a policy, so a runtime
can be restricted to only ever touch its own actor's data.

### Observability

Observability is OpenTelemetry traces landing in CloudWatch, with a
GenAI-specific dashboard on top for stepping through an agent's execution
path. If you already run an OTel collector, you already have the data;
what AgentCore adds is the CloudWatch wiring and the trace viewer, not a
new way of instrumenting anything.

## What's actually new here

Cutting through it: two things don't have an obvious pre-existing AWS
answer. Session-isolated compute built for a *long, stateful* agent turn
rather than a short stateless function, and Gateway's automatic
API/Lambda-to-MCP-tool conversion. Identity's workload-identity concept is
arguably a third, but it's an existing pattern from elsewhere (Kubernetes
and GCP both already have "workload identity") applied to a new kind of
caller, not an invention. Everything else on the list — sandboxed code
execution, a headless browser, a credential broker, a trace pipeline, a
policy engine — already existed somewhere in AWS or in a third-party
product; AgentCore's contribution is putting an agent-sized label and a
per-second meter on each one and shipping them as one console tab.

## What you're actually paying for

Given that, the honest pitch for AgentCore isn't "smarter agents" — none
of these pieces make a model reason better. It's operational: one bill,
one IAM boundary, one observability pipeline, instead of assembling the
same shape yourself from Lambda, Cognito, an OTel collector, and a
third-party browser sandbox. That's a legitimate reason to buy it if
you're already committed to AWS and don't want to own the integration
work. It's a much weaker reason if the pitch you heard was about the
agents themselves — the intelligence lives entirely in whatever model you
plug in, and AgentCore has nothing to do with that part.

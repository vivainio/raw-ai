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

**Runtime** is the core: a serverless microVM per session, billed at
$0.0895 per vCPU-hour and $0.00945 per GB-hour, per-second granularity,
1-second minimum. That's the same execution model Lambda has used for
years (Firecracker microVMs), retuned for a workload Lambda handles
badly — an agent turn that might run for minutes, hold state across tool
calls, and not fit a 15-minute hard ceiling. If you've ever tried to wedge
a LangGraph loop into Lambda and hit that ceiling, Runtime is the answer
to that specific problem, not a new idea.

You still bring your own container. Build a Docker image (ARM64 only —
build it on x86 and dependencies fail silently at runtime), expose
`POST /invocations` and `GET /ping` on port 8080, push it to ECR. Runtime
pulls that image into the microVM per session. This isn't an escape hatch
for "custom environments" — it's the default shape of every agent you
deploy to it. What AWS is selling isn't a way to avoid packaging your own
container; it's what happens to that container once it's running:
per-session isolation, fast cold starts, and a bill measured in vCPU- and
GB-seconds instead of a fixed instance.

**Browser** and **Code Interpreter** bill at the *exact same* vCPU-hour
and GB-hour rates as Runtime. That's the tell — they're not bespoke
products, they're Runtime with a different container image and a product
name, competing directly with things like Browserbase, E2B, or
Cloudflare's Sandbox SDK. The pitch is the managed sandbox and the AWS IAM
boundary around it, not novel execution technology.

**Gateway** is the one piece that saves real work: point it at an OpenAPI
spec or a Lambda function and it generates an MCP-compatible tool
endpoint, with auth, without you writing a tool wrapper by hand. If you
were going to hand-roll "expose fifteen internal APIs to an agent as
tools," Gateway is a genuine shortcut, not a rebrand.

**Identity** doesn't replace Cognito, Okta, or Auth0 — it sits in front of
them as a credential broker, handing agents scoped, short-lived tokens
instead of static API keys. Useful, but it's the same job STS already
does for humans, extended to cover a caller that happens to be a model.

**Memory** is priced per raw event (short-term) and per processed/stored
record plus retrieval call (long-term) — which is to say, a managed table
plus an extraction step, not a new memory architecture. Compare it to the
[agent memory](../agent-memory/index.md) a coding agent keeps in a flat
file on disk: same idea, AWS's version just charges per write and per
read.

**Observability** is OpenTelemetry traces landing in CloudWatch. If you
already run an OTel collector, you already have this; AgentCore's version
just wires it to CloudWatch by default and adds an agent-shaped UI on top
of the trace data.

## What's actually new here

Cutting through it: two things don't have an obvious pre-existing AWS
answer. Session-isolated compute built for a *long, stateful* agent turn
rather than a short stateless function, and Gateway's automatic
API/Lambda-to-MCP-tool conversion. Everything else on the list — sandboxed
code execution, a headless browser, a credential broker, a trace
pipeline, a policy engine — already existed somewhere in AWS or in a
third-party product; AgentCore's contribution is putting an agent-sized
label and a per-second meter on each one and shipping them as one console
tab.

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

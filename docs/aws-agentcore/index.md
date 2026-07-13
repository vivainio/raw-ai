---
icon: lucide/cloud
---

# Hosting: AWS AgentCore

Amazon Bedrock AgentCore is marketed as "an agentic platform for building,
deploying, and operating highly effective agents securely at scale." That
sentence is true and says almost nothing. What matters more than AWS's
label for it is what you actually get for the bill, and how much of it is
new versus an existing AWS primitive wearing an agent-shaped label.

## What ships under the name

AgentCore is a bundle of a dozen separately billed pieces, and the list
has grown since launch:

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

## Taking each piece apart

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
pulls that image into the microVM per session, and that's true of every
agent deployed here, not just an escape hatch for "custom environments."
Packaging your own container is simply required. What you're paying AWS
for is what happens to that container once it's running: per-session
isolation, fast cold starts, and a bill measured in vCPU- and GB-seconds
instead of a fixed instance.

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

### Session isolation

"Session-isolated" gets used constantly in AgentCore's docs without ever
being unpacked, so it's worth pinning down. A session is one continuous
back-and-forth with an agent, identified by a `runtimeSessionId` the
caller supplies. The first invocation with a new session ID gets a
dedicated microVM — its own compute, memory, and filesystem, not shared
with any other session. Later invocations using that same session ID land
on that *same* microVM, so whatever the agent process is holding in
memory is still there — that continuity is what lets a session behave
like an actual ongoing conversation instead of a stateless request every
time. The session ends — on 15 minutes of inactivity by default, a hard
cap of 8 hours, an explicit stop call, or a failed health check — and the
whole microVM is torn down and its memory wiped. A new session, even for
the same user calling the same agent, always gets a fresh one.

The point of contrast is Lambda's warm-start model, which also reuses
execution environments but opaquely, as an efficiency detail you're not
meant to depend on — a warm container can quietly serve unrelated
invocations, and there's no caller-facing guarantee about which one lands
where. AgentCore makes that boundary explicit and guaranteed instead of
incidental: this session ID always maps to its own VM, no other session
ever shares it, and teardown is deliberate rather than left to whatever
the scheduler feels like reusing. That matters more here than it does for
a typical Lambda function because agent sessions often hold things you
don't want leaking across users — executed code, browser cookies and
local storage, accumulated conversation state — where a shared warm
container would be a real security bug, not just a caching quirk.

That isolation applies to the filesystem specifically, and by default it
cuts both ways: a file a session writes exists only on that session's
microVM, invisible to every other session, *and* it doesn't outlive the
VM either — when the session ends, the filesystem goes with it. Invoke
the same session ID again later and you get a clean disk in a new VM, not
your file back. There are two named, opt-in exceptions, both things you'd
have to deliberately configure rather than stumble into: managed session
storage (preview) lets one specific session's files survive a stop/resume
cycle, still scoped to that one session; and a shared filesystem mount
(S3 Files or EFS) breaks isolation on purpose, so multiple sessions,
agents, or outside applications can read and write the same files. Left
at the defaults, there's no accidental way for one session to see another
session's disk.

### Browser

Browser is metered exactly the same way as Runtime — same unit, same
rate, which gives away what it actually is: Runtime wearing a different
container image and a product name, wrapping a managed headless browser
(Playwright- and BrowserUse-compatible) with a live view for watching a
session and a recording for replaying one later. One genuinely specific
feature sits on top: it can cryptographically sign outbound HTTP requests
so a site sees a legitimate, attributable client instead of generic
headless-Chrome traffic, which cuts down on CAPTCHA challenges. Everything
else about it — the sandbox, the isolation, the metering — is Runtime.
It competes directly with Browserbase, E2B, or Cloudflare's Sandbox SDK,
and the pitch is the managed sandbox plus the AWS IAM boundary around it,
not novel execution technology.

Concretely, it's a CDP WebSocket endpoint, and vanilla Playwright
connects to it the normal way:

```python
from bedrock_agentcore.tools.browser_client import browser_session
from playwright.sync_api import sync_playwright

with browser_session(region) as client:
    ws_url, headers = client.generate_ws_headers()
    with sync_playwright() as pw:
        browser = pw.chromium.connect_over_cdp(ws_url, headers=headers)
        page = browser.contexts[0].pages[0]
        page.goto("https://example.com")
```

`connect_over_cdp` is the same call any browser-automation tool uses to
attach to an existing Chrome instance. The model's driving loop — read
the page, decide an action, click or fill or navigate, look again —
doesn't change at all; only the target does, a remote AWS-managed
Chromium instead of a local subprocess. The one real wrinkle is that
`headers` argument: it carries SigV4-signed AWS auth from
`generate_ws_headers()`, not just a WebSocket URL you can open from
anywhere. A tool that only knows plain CDP, with no way to attach a
signed header to the handshake, can't just point at this endpoint and
go — it has to know how to sign the request first. From Playwright's
side this is a drop-in remote browser; from a generic CDP client's side,
it's gated behind AWS auth rather than an open localhost port.

Worth spelling out what "the model drives the browser" actually means
mechanically, since it's easy to picture the model writing Playwright
code directly. Normally it doesn't. It's the same `tool_use`/`tool_result`
loop from the [Agent Dialogue](../agent-dialogue/index.md) chapter,
applied to a page instead of a shell, with the harness doing the actual
Playwright calls:

1. The harness takes a **snapshot** of the current page — a compact text
   tree of interactive elements, each with a short reference like
   `ref=e3` — not a screenshot. That snapshot comes back as the
   `tool_result`.
2. The model reads that text, decides "click the login button," and
   emits a `tool_use` block: `{"name": "click", "input": {"ref": "e3"}}`.
3. The harness, not the model, turns `ref=e3` into an actual Playwright
   call against the CDP connection above, and runs it.
4. The harness takes a **new** snapshot of whatever the page looks like
   now and returns that as the next `tool_result`.
5. Repeat — one tool call, one action, one updated snapshot, every turn.

Playwright is running the whole time, but the model only ever sees the
snapshot/ref layer, never raw HTML or the Playwright API itself. A
different pattern is possible on the same CDP connection: hand the model
Code Interpreter instead, and it can write a whole multi-step Playwright
script and run it in one shot, trading the ability to react after each
click for fewer round-trips. AgentCore doesn't pick between these —
Browser is just the raw CDP endpoint; which loop you get depends on
whatever harness is wrapping it, not on AWS.

### Code Interpreter

Code Interpreter is the same story again: another container image on the
same Runtime substrate, this one pre-loaded with Python, JavaScript, and
TypeScript kernels instead of a browser. Same metering, same per-session
isolation. If Runtime, Browser, and Code Interpreter feel like
three products, that's the packaging talking — underneath they're one
execution primitive wearing three skins.

It's easy to conflate this with Runtime itself, so it's worth separating
what each one holds. Runtime hosts *your* code — the agent process you
wrote and deployed. Code Interpreter is a separate sandbox your agent
calls into as a tool, specifically for running code the *model* just
generated, that you don't want anywhere near the process holding your
agent's own state and credentials:

```python
from bedrock_agentcore.tools.code_interpreter_client import CodeInterpreter

code_client = CodeInterpreter("us-west-2")
code_client.start()
response = code_client.invoke("executeCode", {
    "language": "python",
    "code": "print(2 + 2)",
})
code_client.stop()
```

The model produces a code string, the tool handler ships it to a Code
Interpreter session, and whatever comes back on stdout is what the model
sees as the tool result — the same shape as a coding agent's own
bash/Python execution, except the sandbox is a rented AWS resource behind
an API call rather than a subprocess on the same machine.

That last comparison is worth taking seriously, because it's the actual
reason this tool exists rather than being a footnote. The scope is bigger
than arithmetic — any data analysis where the data is too big or too
regular to push through the model's own reasoning reliably: filtering
ten thousand rows down to the ones that match, aggregating a column,
diffing two large blobs, reshaping a file from one format to another.
Doing that by having the model read the whole thing and reason it out
token by token is slow, expensive in context, and error-prone in a way
running actual code isn't — code either produces the right filtered rows
or it doesn't, there's no probabilistic guessing in between. The reliable
move is always the same: have the model write a short, deterministic
script, run it, and read back the (usually much smaller) result instead
of asking the model to hold and manipulate the raw data itself. This is
exactly what's happening on your own machine constantly when Claude Code
reaches for a shell command or a quick Python script to grep a log,
reshape a CSV, or count matches, rather than eyeballing all of it
directly — same move, just a local subprocess instead of a rented AWS
sandbox. Code Interpreter just gives a cloud-hosted agent the same
"delegate the data work to real code" escape hatch a local coding agent
already has for free — the technique predates the product.

It's also a separate disk, not a shared mount with the Runtime container
that invoked it. A file your agent process can see isn't automatically
visible to code running inside a Code Interpreter session — moving
something across that boundary is an explicit `writeFiles` /
`readFiles` call, same as it would be with any remote sandbox:

```python
# push a file into the sandbox
code_client.invoke("writeFiles", {
    "content": [{"path": "data.csv", "text": "a,b\n1,2\n3,4\n"}]
})

# run code that reads it from the sandbox's own disk
code_client.invoke("executeCode", {
    "language": "python",
    "code": "print(open('data.csv').read())",
})
```

That `print()` is why the last call needed nothing further: stdout rides
back on the same response that ran the code, as part of the `executeCode`
result — `event["result"]["content"]` holds it, no separate fetch
required. A file the code *writes to disk* instead of printing is a
different case — that never appears in the `executeCode` response at
all, since it stayed on the sandbox's filesystem. Getting it out is the
same explicit pull `writeFiles` needed on the way in:

```python
code_client.invoke("executeCode", {
    "language": "python",
    "code": "open('out.csv', 'w').write('x,y\n1,2\n')",
})
code_client.invoke("readFiles", {"paths": ["out.csv"]})
```

So: printed output comes back for free with the call that produced it;
anything written to disk needs its own `readFiles`, same as it needed its
own `writeFiles` going in. Nothing is shared implicitly either way —
every hop across the boundary is a call your code makes on purpose, the
same pattern as Memory's `create_event`/`retrieve_memories` pair: two
separate stores, and you're the one wiring the pipe between them.

### Gateway

Gateway is the one piece here that saves real work. Point it at an
OpenAPI spec, a Lambda function, or a Smithy model and it stands up an
MCP server for you — one tool per API operation, with the tool's name and
input schema generated straight from the spec — without you writing an
MCP server or a tool wrapper by hand. An agent talks to the result the
same way it talks to any MCP server: connect, `list_tools()`,
`call_tool()`.

```python
target_config = {"mcp": {"openApiSchema": {"s3": {"uri": openapi_s3_uri}}}}
credential_config = [{
    "credentialProviderType": "API_KEY",
    "credentialProvider": {"apiKeyCredentialProvider": {
        "credentialParameterName": "api_key",
        "providerArn": credential_provider_arn,
        "credentialLocation": "QUERY_PARAMETER",
    }},
}]
gateway_client.create_gateway_target(
    gatewayIdentifier=gateway_id,
    targetConfiguration=target_config,
    credentialProviderConfigurations=credential_config,
)
```

That call is doing two separate things, and "inbound and outbound auth
wired in" glosses over which is which. Inbound auth is set once, on the
gateway itself, not per target — a JWT authorizer pointing at whatever
OIDC provider issues the agent's token, gating who's allowed to call the
gateway's MCP endpoint at all. Outbound auth is set per target, in the
`credentialProviderConfigurations` block above — the API key, OAuth
token, or IAM role the gateway attaches when it forwards a call through
to the actual backend API. The agent authenticates once, to the gateway;
it never sees the downstream credential. That's the same inbound/outbound
split Identity uses further down, just applied to the gateway's own
outbound calls instead of the agent's.

The search half works the same way as any other tool: a gateway created
with search enabled stands up a vector store behind the scenes, embeds
every attached tool's name and description into it, and exposes one more
MCP tool, `x_amz_bedrock_agentcore_search`, that takes a `query` string
and returns whichever tools match. An agent sitting in front of hundreds
of tools calls that search tool first, gets back the handful that are
relevant, and only those descriptions ever land in context — instead of
every tool description being loaded up front regardless of whether that
turn needs it.

Turning an OpenAPI spec into callable tools isn't itself an AWS-only
trick — several self-hosted libraries do that same conversion for free,
so "spec in, tools out" alone doesn't justify Gateway's existence. What a
local converter doesn't hand you is the other two pieces: the downstream
API credential lives in the gateway's credential provider, not in the
agent's own process or context, so a compromised or misbehaving agent
can't leak or misuse it directly; and the search tool is a managed vector
index you didn't have to stand up and keep in sync as tools get added or
removed. Skip both — a handful of tools, agent already trusted with the
API key — and a local OpenAPI-to-MCP converter gets most of the value
with one less service to operate. Gateway earns its keep specifically
when the tool count or the credential-isolation requirement gets large
enough that those two gaps start to matter.

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

Given that, the honest pitch for AgentCore is operational, not "smarter
agents" — none of these pieces make a model reason better. One bill,
one IAM boundary, one observability pipeline, instead of assembling the
same shape yourself from Lambda, Cognito, an OTel collector, and a
third-party browser sandbox. That's a legitimate reason to buy it if
you're already committed to AWS and don't want to own the integration
work. It's a much weaker reason if the pitch you heard was about the
agents themselves — the intelligence lives entirely in whatever model you
plug in, and AgentCore has nothing to do with that part.

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

Worth unpacking what "attaches" means, since it's easy to read that as
just plugging a stored string into a template. The `providerArn` in the
target config above doesn't point at the secret directly — it points at a
**credential provider**, a named object in AgentCore Identity (the same
piece described below), created once, upfront:

```python
response = identity_client.create_api_key_credential_provider(
    name="nasa-api-key",
    apiKey=nasa_api_key,   # the real secret, given exactly once
)
credential_provider_arn = response["credentialProviderArn"]
```

That call stores the key encrypted in AWS Secrets Manager — Identity
doesn't reimplement secret storage, it wraps it — and hands back an ARN.
From then on the ARN is the only thing that appears anywhere: in the
target config, in logs, anywhere. The literal key value never does.

When an agent calls a gateway tool, the gateway matches the call to its
target, resolves that target's credential-provider ARN against Identity,
and injects the result at whichever location the target declared
(`QUERY_PARAMETER` / `api_key` in the example above; a header for others).
For a plain API-key provider that resolution is a straight Secrets
Manager fetch. Swap in an OAuth2 provider instead and it's more than a
fetch: Identity runs an actual grant against a token endpoint, caches the
resulting access token, and refreshes it before it expires — active
protocol work, not a passive lookup.

Both of those are still a single shared identity, though — every caller
hitting that target goes out to the backend as the same key or the same
OAuth client, and the backend has no way to tell them apart. Genuine
per-user access needs a third shape: an OAuth provider with
`grantType: TOKEN_EXCHANGE` and `on_behalf_of`, which takes the *caller's
own inbound JWT* and exchanges it for a downstream token scoped to that
specific user, rather than attaching a fixed credential at all. That only
works when inbound auth to the gateway is JWT-based to begin with (there
has to be a caller identity to exchange) and the backend itself supports
token exchange (RFC 8693) — an API-key-only backend has no concept of
"which user," so there's nothing to retrofit per-user access onto no
matter how the gateway target is configured.

It's worth being honest about what this buys over just putting the key in
Secrets Manager or SSM Parameter Store directly and having the agent fetch
it. Storage-wise, nothing — that's literally where the API-key provider
puts it. The difference is where the fetch-and-attach happens. Fetch it
yourself and the plaintext key passes through the agent's own process —
the same process handling the conversation and running model-directed
tool code — if only for the moment between the `GetSecretValue` call and
attaching it to a request; a compromised or misdirected agent is a real
path for that value to leak. Route it through Gateway and the agent never
receives the key at all — the fetch and the injection both happen on the
gateway's side of the hop, at a location pinned by the target config
rather than whatever a handler happens to do with a value it holds. For a
static key, that's the entire gap: not better storage, just one less
process the secret ever has to pass through. For OAuth token exchange, the
gap is bigger, because Secrets Manager has no equivalent of a grant flow
at all — that part is real protocol work Gateway is doing for you, not
storage with extra steps.

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

Start with the obvious question: an agent already runs under an IAM role —
Runtime's execution role, no different from any Lambda function's. Why
does AgentCore need a separate identity piece at all? Because IAM's reach
stops at the AWS boundary. Every permission an IAM policy grants is
scoped to an AWS resource — a bucket, a table, another function —
evaluated by AWS's own policy engine against AWS's own resource ARNs. The
moment an agent needs to call something outside AWS-hosted services — a
third-party SaaS API, a customer's OAuth-protected endpoint — there's no
policy grammar for that at all: no way to express "attach this OAuth
token" or "act as this specific end user downstream." Identity exists to
cover exactly that territory, not to extend or replace IAM inside the
boundary IAM already governs.

Inbound auth — who's allowed to call the agent in the first place — stays
inside the boundary IAM already covers: SigV4 or an OAuth JWT gates the
call the same way IAM or a JWT authorizer gates any other API. Outbound
auth is the new territory, and it's where a **workload identity** shows
up: a primitive distinct from any IAM role or Cognito user, representing
the agent itself as a caller with its own ARN, holding a token vault for
reaching things outside AWS. That vault covers three shapes — OAuth
authorization-code grant for user-delegated access, client-credentials
grant for service-to-service calls, and on-behalf-of token exchange
(RFC 8693), which trades the caller's own inbound JWT for a downstream
token scoped to that specific user, so the agent can act for someone
without re-prompting them for consent.

Identity doesn't replace Cognito, Okta, or Auth0 either — end users still
authenticate through whichever of those already runs. What it adds is the
missing link between "a user logged in somewhere else" and "the agent can
now call a downstream API as that user," a link IAM was never built to
hold because the far end of that call was never an AWS resource. Most
clouds already have some version of a workload-identity concept for
service-to-service calls — Kubernetes and GCP both ship one — so
AgentCore's version is that pattern extended to a caller that happens to
be a model, not a wholly new idea; the RFC 8693 exchange piece is the
part that's genuinely specific to bridging an inbound agent call to an
outbound per-user one.

None of what follows is worth worrying about if the backend you're calling
doesn't speak OAuth in the first place. A downstream API that only takes a
static key skips all of it — that's the API-key credential provider from
Gateway, a straight secret fetch with no discovery document, no grant
type, no consent flow. The token vault's OAuth2 machinery only matters
once the backend is itself a real OAuth2/OIDC authorization server, and
even then it's tiered: plain client-credentials (`M2M`) or user
authorization-code (`USER_FEDERATION`) is what most OAuth2 implementers
ship, while on-behalf-of token exchange (RFC 8693) is a narrower ask
plenty of providers never implement — without it, per-user delegation
falls back to running the full consent flow separately for every user
rather than the single-exchange shortcut.

Mechanically, the agent doesn't call the token vault's API directly — it
decorates whatever function needs the credential, and the SDK resolves it
before that function body runs:

```python
from bedrock_agentcore.identity import requires_access_token

@requires_access_token(
    provider_name="google-provider",
    scopes=["https://www.googleapis.com/auth/drive.metadata.readonly"],
    auth_flow="USER_FEDERATION",
    on_auth_url=lambda url: print(f"Authorize here: {url}"),
)
async def read_drive_file(*, access_token: str):
    # access_token is already resolved by the time this line runs
    ...
```

Underneath that one decorator sit three separate calls the SDK makes on
your behalf — `CreateWorkloadIdentity`, `GetWorkloadAccessToken`,
`GetResourceOauth2Token` — none of which the agent code ever touches.
Which of those calls needs a human depends on `auth_flow`: `M2M` returns
the access token in one round trip, no user involved, the same shape as
the API-key case from Gateway's credential providers. `USER_FEDERATION` is
the three-legged case — the first call comes back with an authorization
URL instead of a token, the decorator hands that URL to `on_auth_url`
(print it, stream it into a chat UI, push it to a webhook — whatever the
app does with a "please consent" link), then polls
`GetResourceOauth2Token` until the user finishes consenting and a token
appears. AgentCore stores the provider's refresh token alongside that
first token and uses it silently on every later call, so the human is only
in the loop once, not on every downstream request.

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

### Harness

Every piece documented so far is something your own agent code calls —
Runtime hosts that code, and Gateway, Memory, Browser, Code Interpreter,
and Identity are APIs it reaches into from inside the loop you wrote.
Harness removes that loop from your code entirely. `CreateHarness` takes a
model, a system prompt, a tool list, and a memory config as one API call;
`InvokeHarness` runs whatever loop AWS assembled from that config and
streams the result back. There's no framework decision to make, because
the harness *is* one — specifically [Strands Agents](https://strandsagents.com),
AWS's own open-source agent framework, wired up from your config and run
inside the same session-isolated Runtime microVM documented above
(CloudTrail even records harness calls under `AWS::BedrockAgentCore::Runtime`).

```python
control_client.create_harness(
    harnessName="research-agent",
    executionRoleArn=execution_role_arn,
)
# poll get_harness until status == "READY", then invoke:

response = data_client.invoke_harness(
    harnessArn=harness_arn,
    runtimeSessionId=session_id,
    messages=[{
        "role": "user",
        "content": [{"text": "Research three tropical vacation options under $3k."}],
    }],
)
for event in response["stream"]:
    if "contentBlockDelta" in event:
        print(event["contentBlockDelta"]["delta"].get("text", ""), end="")
```

That's the entire integration surface: no `BedrockAgentCoreApp` wrapper, no
container to build and push to ECR, no tool-calling loop to write by
hand — all three of which Runtime still requires. The CLI path
(`agentcore create`, `agentcore deploy`, `agentcore invoke`) reaches the
same config and gets a scaffolded project instead of raw API calls, but
it's the same trade either way.

Which makes pydantic.ai the wrong comparison. Pydantic.ai, like Strands
itself, LangGraph, or CrewAI, is a library you import and write Python
against — a real reduction in boilerplate, but you still own the loop and
ship code that calls it. Harness's fork is a level up from that: code
versus config. Reach for pydantic.ai and you've already decided to write
an agent in Python; reach for Harness and you haven't written anything,
you've filled in a form. The nearer comparison is a hosted agent-builder
or something like OpenAI's Assistants/Responses API, not another Python
package.

That config-not-code trade shows up concretely in what's a field versus a
redeploy. Switching model provider — Bedrock, OpenAI, Gemini, or anything
LiteLLM-compatible — is a config value, and doing it mid-session without
losing the conversation is the same field, not a client swap. Attaching an
AWS Skill from Git, S3, or the curated catalog is a toggle. Wiring in
Memory, Gateway, Browser, or Code Interpreter is a config block instead of
the `import` and explicit client calls each of those needed everywhere
else in this chapter. None of the mechanics underneath change because
Harness sits in front of them: same session-isolated microVM, same
actor-scoped memory namespaces, same credential-provider indirection in
Gateway. There's no separate meter for Harness either — a session bills
for whatever mix of Runtime, Memory, Gateway, and the rest it actually
used, same as if you'd called them yourself.

The trade runs out exactly where "no code" has to. There's no choice of
agent framework — you get Strands, full stop — no bidirectional
streaming, no non-loop patterns like a graph or workflow, no hooks into
the loop's internals. AWS's own docs treat that boundary as expected
rather than a gap to apologize for: export the harness to Strands code
and keep running it on Runtime once config stops being enough, so the
managed layer is something you can grow out of instead of a wall you hit.

## What's actually new here

The fairer test isn't whether some third party already ships something
adjacent — Browserbase, E2B, and Mem0 all predate their AgentCore
counterparts — it's whether AWS itself had an answer before AgentCore
shipped. On that bar, most of this list clears: Runtime's session-isolated
long-running compute, Gateway, Memory, Browser, Code Interpreter, and
Harness are all things AWS is standing up for the first time, not a new
name on an existing AWS service. Harness is the clearest case — a
managed, codeless agent loop didn't exist anywhere in AWS's lineup before
this.

The exceptions are the pieces built on AWS tech that already shipped
under a different label. Observability's dashboard is new; the pipeline
underneath — OTel traces into CloudWatch — is the same one AWS already
ran. Policy reuses Cedar, the engine already behind Verified Permissions,
for a new question — which tools can this agent call — without a new
engine underneath. Evaluations/Optimization extends Bedrock's existing
model-evaluation tooling to score agent traces instead of raw model
outputs — same capability, a wider target. Identity sits in between: the
workload-identity primitive is new for AWS, but the pattern isn't new
anywhere — Kubernetes and GCP already had "workload identity" — so this
is AWS filling a gap in its own platform rather than inventing the idea.

That leaves a different verdict than "AWS relabeled a pile of existing
services": a few pieces genuinely are that, but most of AgentCore is AWS
building an agent stack it simply didn't have, one piece at a time,
priced and documented as a dozen separate line items rather than shipped
as one coherent idea.

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

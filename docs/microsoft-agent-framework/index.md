---
icon: lucide/git-merge
---

# Frameworks: Microsoft Agent Framework

Microsoft Agent Framework (MAF) is what happened after Microsoft spent a
couple of years running two separate agent frameworks in parallel —
Semantic Kernel, the enterprise-plugin-and-planner one, and AutoGen, the
multi-agent-conversation research one — and merged them into a single
open-source project with migration guides off both. It's `pip install
agent-framework` and write Python, the same shape as
[Pydantic AI](../pydantic-ai/index.md) and
[Strands](../strands-agents/index.md), not a hosted service. The Python
package ships alongside a .NET package and, more recently, a Go one, all
built against the same concepts — which matters less for what you write
day to day than for the fact that "agent," "thread," and "workflow" mean
the same thing regardless of which language's docs you're reading.

The model connection is a swappable `ChatClient` — `OpenAIChatClient`,
`FoundryChatClient` for Azure AI Foundry, others for Anthropic and Ollama
— so despite Microsoft's own samples defaulting to Foundry, the framework
itself isn't locked to Azure any more than Strands is locked to Bedrock.

## The core shape: Agent, a chat client, run or stream

An `Agent` takes a client and a system prompt, and tools go in the
constructor as plain `@tool`-decorated functions — docstring becomes the
description, type hints become the input schema, the same deal as
Pydantic AI's `@agent.tool` and Strands' `@tool`:

```python
from typing import Annotated
from agent_framework import Agent, tool
from agent_framework.openai import OpenAIChatClient

@tool
def get_weather(location: Annotated[str, "The city and state, e.g. Seattle, WA"]) -> str:
    """Get the current weather for a given location."""
    return fetch_weather(location)

agent = Agent(
    client=OpenAIChatClient(),
    name="WeatherAgent",
    instructions="You are a helpful weather assistant.",
    tools=[get_weather],
)
result = await agent.run("What's the weather in Seattle?")
print(result.text)
```

Streaming is the same method with `stream=True` rather than a separate
`run_stream`, and the object it returns — a `ResponseStream` — supports
iterating it for real-time output, awaiting `get_final_response()` to
skip straight to the aggregate, or both in sequence, since the second call
reuses whatever the iteration already collected instead of re-running
anything.

## Statefulness sits between Pydantic AI's and Strands'

Calling `agent.run()` twice in a row doesn't remember the first call —
same starting point as [Pydantic AI's "no session
object"](../pydantic-ai/index.md#theres-no-session-object). Where MAF
diverges is `AgentSession`, an object the caller creates once and then
passes into every `run()` call for that conversation:

```python
session = agent.create_session()

await agent.run("My name is Alice and I love hiking.", session=session)
result = await agent.run("What do you remember about me?", session=session)
```

That's less than Strands' `SessionManager`, which ships ready-made
backends — `S3SessionManager`, `AgentCoreMemorySessionManager` — that
persist every turn to a specific store with zero extra code. MAF's own
hosting docs show the gap directly: a custom request handler keeps
sessions in a plain `dict[str, AgentSession]` keyed by session ID, with an
explicit warning that this in-memory store is lost on restart and
production deployments need to swap in something durable themselves. The
session object exists and carries conversation state across calls —
that's real, and it's more than Pydantic AI gives you — but persisting it
past the process's lifetime is still the caller's problem, not a
constructor argument away the way it is in Strands.

## Multi-agent orchestration: named patterns on top of one graph engine

This is where MAF stops looking like Pydantic AI or Strands. Strands
offers two multi-agent objects — `Swarm` for handoff routing, `Graph` for
an explicit DAG. MAF instead ships one general graph runtime,
`WorkflowBuilder`, plus five pre-built `Builder` classes that wire common
orchestration shapes on top of it: `SequentialBuilder`, `ConcurrentBuilder`,
`HandoffBuilder`, `GroupChatBuilder`, and `MagenticBuilder` (a manager
agent dynamically routing to specialist agents, the pattern AutoGen was
best known for). Sequential is the simplest — agents run in order, each
one seeing the previous agent's output:

```python
from agent_framework.orchestrations import SequentialBuilder

writer = chat_client.as_agent(instructions="Write a punchy marketing sentence.", name="writer")
reviewer = chat_client.as_agent(instructions="Give brief feedback on the previous message.", name="reviewer")

workflow = SequentialBuilder(participants=[writer, reviewer]).build()
events = await workflow.run("Write a tagline for a budget-friendly eBike.")
```

Drop below the named patterns and `WorkflowBuilder` is the same
executors-and-edges shape as Strands' `GraphBuilder`:

```python
from agent_framework import WorkflowBuilder

builder = WorkflowBuilder(start_executor=processor)
builder.add_edge(processor, validator)
builder.add_edge(validator, formatter)
workflow = builder.build()
```

The part neither Strands' `Graph` nor Pydantic AI's plain `await` calls
document is what actually drives execution underneath: a modified
[Pregel](https://kowshik.github.io/JPregel/pregel_paper.pdf) model,
running in synchronous rounds called supersteps. Each superstep collects
every message queued by the previous round, routes them to their target
executors, runs all of those targets concurrently, then blocks until
every one of them finishes before the next superstep starts. That barrier
is what makes a workflow's execution order deterministic given the same
input — useful on its own, and the reason checkpointing (saving state at
superstep boundaries, for recovery after a crash) is something the
framework can offer as a built-in rather than something you'd have to
design around a graph with no defined execution boundaries at all.

## Pausing for approval: the same content-type sentinel at two levels

A `@tool` can be marked `approval_mode="always_require"`. When the model
calls it, the run doesn't execute the function — it stops and returns a
`function_approval_request` in the response's `user_input_requests`,
mirroring [Pydantic AI's `ApprovalRequired` →
`DeferredToolRequests`](../pydantic-ai/index.md#pausing-without-a-workflow-engine-deferred-tools)
and [Strands' `BeforeToolCallEvent.interrupt()`](../strands-agents/index.md#pausing-for-approval-hooks-and-interrupts):

```python
@tool(approval_mode="always_require")
def delete_file(path: str) -> str:
    """Delete a file."""
    return f"deleted {path}"

result = await agent.run("Delete config.yaml")
for req in result.user_input_requests:
    print(f"Approve {req.function_call.name}({req.function_call.arguments})?")
    approved = input("y/n: ").lower() == "y"
    approval_message = Message(role="user", contents=[req.to_function_approval_response(approved)])
    result = await agent.run([approval_message], session=session)
```

The distinctive move is that the same sentinel works one level up, inside
a `SequentialBuilder`/`WorkflowBuilder` run. An approval-required tool
called by any agent in the pipeline pauses the whole workflow and emits a
`request_info` event carrying that identical `function_approval_request`
content, which a caller answers by feeding a
`{request_id: to_function_approval_response(...)}` dict back into
`workflow.run(responses=...)`. One mechanism covers a single agent
pausing on a tool call and a five-agent pipeline pausing on one of its
participants' tool calls — Strands' hook system and Pydantic AI's
deferred tools are both scoped to a single agent's run, with no
equivalent built-in for "the whole graph waits."

## Deploying: Foundry Hosted Agents, the same house-vendor pattern as AgentCore

Microsoft runs a managed host for this framework the way AWS runs
AgentCore Runtime for Strands — the same company builds the open-source
framework and sells the platform that runs it in production. Foundry
Hosted Agents packages an `Agent` or workflow as a container and exposes
it through one of two protocols: **Responses**, an OpenAI-compatible
`/responses` endpoint where the platform manages conversation history and
streaming for you, or **Invocations**, where you write the HTTP handler
yourself:

```python
from agent_framework import Agent
from agent_framework.foundry import FoundryChatClient
from agent_framework_foundry_hosting import ResponsesHostServer

agent = Agent(
    client=FoundryChatClient(project_endpoint=endpoint, model=model, credential=credential),
    instructions="You are a helpful AI assistant.",
    default_options={"store": False},  # platform already persists history
)

ResponsesHostServer(agent).run()
```

`store=False` there is the tell: Responses-protocol hosting already does
what `AgentSession` above made the caller's job, the same trade Strands'
`AgentCoreMemorySessionManager` makes against hand-written persistence —
pick the platform's managed conversation store over rolling your own, but
only once you're deployed to that platform specifically. Outside Foundry
hosting, nothing about `Agent`, `WorkflowBuilder`, or the approval
sentinel is Azure-specific; a Foundry-hosted deployment and a plain
`OpenAIChatClient` agent running in your own container use the identical
framework code, the same way a Strands agent doesn't know or care whether
it eventually lands on AgentCore Runtime.

## Where this sits relative to Pydantic AI and Strands

All three are `pip install`-and-own-the-process frameworks, so the real
differences are in what ships built-in versus what you wire up. Pydantic
AI's bet is static typing — `Agent[DepsT, OutputT]` generics catching a
mismatched dependency at type-check time — with no orchestration object
at all; "another agent" is just an `await` in a tool. Strands sits in the
middle: built-in session backends, two multi-agent objects, hook-based
interrupts. MAF goes furthest toward naming the shapes explicitly —
five built-in orchestration patterns instead of two, a documented
execution model (supersteps) underneath the graph instead of an
undocumented one, and an approval mechanism that scales from one agent to
an entire pipeline without changing shape. The cost of that is real too:
more vocabulary to learn before writing anything (`AgentSession`,
`Executor`, `WorkflowBuilder`, five different `*Builder` classes,
`RequestInfoExecutor`), and it's the newest of the three, carrying visible
scar tissue from merging two frameworks that made different choices —
Semantic Kernel's planner-and-plugin vocabulary and AutoGen's
conversation-centric one both still show through in different corners of
the API.

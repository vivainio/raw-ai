---
icon: lucide/waypoints
---

# Frameworks: Strands Agents

Strands Agents is AWS's own open-source agent framework — the exact
thing running underneath AgentCore's Harness when you configure a
harness with no code at all. Set Harness aside, though, and using
Strands directly is the same shape as [Pydantic AI](../pydantic-ai/index.md):
`pip install`, write Python, own the process it runs in. The interesting
comparison isn't "which one is more AWS" — it's how much of the surface
each one hands you as a built-in object versus something you wire up
yourself.

## The core shape: Agent, tools, direct calls

An `Agent` takes tools and a system prompt in its constructor, and the
agent object itself is callable — there's no `.run()`/`.run_sync()` split
the way Pydantic AI has:

```python
from strands import Agent, tool

@tool
def weather(city: str) -> dict:
    """Get current weather for a city."""
    return fetch_weather(city)

agent = Agent(system_prompt="You are a helpful assistant.", tools=[weather])
result = agent("What's the weather in Seattle?")
```

The `@tool` decorator does the same job as Pydantic AI's `@agent.tool` —
docstring becomes the description, function signature becomes the input
schema — but it isn't bound to one agent at decoration time. A `@tool`
function is a portable callable, handed to whichever `Agent`'s `tools=`
list wants it, which is why it slots into the agents-as-tools pattern
below with no extra wiring.

Structured output is a method call rather than a constructor-time
`output_type`, or a constructor-time default if every call should return
the same shape:

```python
from pydantic import BaseModel
from strands import Agent

class PersonInfo(BaseModel):
    name: str
    age: int

agent = Agent()
result = agent.structured_output(PersonInfo, "John Smith is 30 years old")
```

## Statefulness is a constructor argument, not something you build

This is the sharpest contrast with the [Pydantic AI chapter's "no session
object"](../pydantic-ai/index.md#theres-no-session-object) section.
Strands ships an abstract `SessionManager` base class plus ready-made
backends — `FileSessionManager` for local disk, `S3SessionManager` for
S3, and, the one that matters most for this book,
`AgentCoreMemorySessionManager` wired straight into AgentCore Memory:

```python
from strands import Agent
from strands.session.s3_session_manager import S3SessionManager

session_manager = S3SessionManager(session_id="user-456", bucket="my-agent-sessions")
agent = Agent(session_manager=session_manager)

agent("Tell me about AWS S3")
agent("What did I just ask about?")  # no message_history argument anywhere
```

There's no `message_history` parameter in that second call. The session
manager loads prior turns before the model runs and persists new ones
after, on every call, automatically — the same shape as AgentCore
Harness's auto-persistence, just pluggable to whichever backend you
choose instead of AWS running it unconditionally.

## Multi-agent orchestration is a first-class object

Pydantic AI treats "another agent" as a plain `await` call from inside a
tool — no dedicated abstraction. Strands ships two built-in orchestrator
classes for the same job: `Swarm`, where agents hand off to each other
with routing driven by structured output, and `Graph`, an explicit DAG of
agent or swarm nodes wired together with edges:

```python
from strands import Agent
from strands.multiagent import GraphBuilder, Swarm

research_agents = [
    Agent(name="medical_researcher", system_prompt="..."),
    Agent(name="technology_researcher", system_prompt="..."),
]
research_swarm = Swarm(research_agents)
analyst = Agent(system_prompt="Analyze the provided research.")

builder = GraphBuilder()
builder.add_node(research_swarm, "research_team")
builder.add_node(analyst, "analysis")
builder.add_edge("research_team", "analysis")
graph = builder.build()

result = graph("Research the impact of AI on healthcare")
```

The plain-function version is still there too — a `@tool` that builds and
calls another `Agent` inline is exactly the Pydantic AI pattern:

```python
@tool
def research(query: str) -> str:
    """Research a topic thoroughly."""
    return str(Agent(tools=[search_web])(query))

writer = Agent(tools=[research])
writer("Write a post about AI agents")
```

Both exist side by side. The built-in classes earn their keep when the
handoff/routing/dependency structure is something worth the framework
tracking explicitly — a `maxSteps` cap, an actual DAG with edges — rather
than logic buried inside a tool function's body.

## Pausing for approval: hooks and interrupts

Pydantic AI's answer to "a tool needs a human" is raising
`ApprovalRequired` and getting a `DeferredToolRequests` sentinel back as
the run's output. Strands reaches the same outcome through its hook
system: a `HookProvider` registers a callback on `BeforeToolCallEvent`,
and `event.interrupt(...)`, called from inside that callback, is what
actually pauses the run:

```python
from strands import Agent, tool
from strands.hooks import BeforeToolCallEvent, HookProvider, HookRegistry

@tool
def delete_files(paths: list[str]) -> bool: ...

class ApprovalHook(HookProvider):
    def register_hooks(self, registry: HookRegistry, **kwargs) -> None:
        registry.add_callback(BeforeToolCallEvent, self.approve)

    def approve(self, event: BeforeToolCallEvent) -> None:
        if event.tool_use["name"] != "delete_files":
            return
        approval = event.interrupt("delete-approval", reason=event.tool_use["input"])
        if approval.lower() != "y":
            event.cancel_tool = "User denied permission to delete files"

agent = Agent(hooks=[ApprovalHook()], tools=[delete_files])
result = agent("Delete a/b/c.txt")
while result.stop_reason == "interrupt":
    responses = [
        {"interruptResponse": {"interruptId": i.id, "response": input(f"Approve {i.reason}? (y/N): ")}}
        for i in result.interrupts
    ]
    result = agent(responses)
```

Same shape underneath as Pydantic AI's deferred tools — the run stops
instead of erroring, and resuming means sending something back keyed by
an ID — but the decision point sits in a different place. Pydantic AI's
`ApprovalRequired` is raised from inside the tool itself; Strands'
`BeforeToolCallEvent` fires before `delete_files` ever runs, so
`delete_files` carries no approval logic at all, the hook owns it
entirely. And the resume payload is a plain list of `interruptResponse`
dicts fed straight back into the same callable `agent(...)`, not a
separate `deferred_tool_results` parameter alongside a hand-carried
`message_history` — a session manager, if one's attached, is already
carrying the history, so the interrupt response is the only thing left
for the caller to supply.

## Deploying to AgentCore Runtime

Same `BedrockAgentCoreApp` wrapper documented in the [AWS
AgentCore](../aws-agentcore/index.md) chapter's Runtime section — Strands
plugs in exactly the way any framework does, because Runtime's
integration surface is "a Python function that returns a string or a
stream," not something Strands-specific:

```python
from bedrock_agentcore.runtime import BedrockAgentCoreApp
from strands import Agent

app = BedrockAgentCoreApp()
agent = Agent(system_prompt="Be concise.")

@app.entrypoint
def invoke(payload):
    return str(agent(payload["prompt"]))

if __name__ == "__main__":
    app.run()
```

None of the Strands-specific machinery above — session manager, `Swarm`,
interrupts — is required to reach this point. Runtime doesn't know or
care that this happens to be the same framework Harness runs internally;
from Runtime's side, this container looks exactly like the [Pydantic AI
Runtime example](../aws-agentcore/index.md#runtime).

## Where the AgentCore-specific wiring actually is

Set Runtime aside and the two frameworks converge on the story from the
[Pydantic AI](../pydantic-ai/index.md) chapter: Gateway, Identity,
Browser, and Code Interpreter are plain `bedrock_agentcore`/boto3 calls,
equally manual no matter which framework issued them. What Strands gets
that Pydantic AI doesn't is narrower than a blanket "better AgentCore
integration": `AgentCoreMemorySessionManager` turns on automatic
persistence to AgentCore Memory with a constructor argument instead of
hand-written `create_event`/`retrieve_memories` calls, and Harness
itself — Strands running headless inside a config-driven wrapper AWS
built around this one framework specifically. Outside those two, picking
Strands over Pydantic AI buys built-in session management, `Swarm`/`Graph`
orchestration objects, and a hook-based interrupt system — real
differences, but general framework ergonomics rather than anything
AgentCore-flavored.

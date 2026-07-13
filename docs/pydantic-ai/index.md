---
icon: lucide/shield-check
---

# Frameworks: Pydantic AI

Pydantic AI markets itself as bringing "a FastAPI-like developer
experience to GenAI development." Unlike AgentCore, there's no bundle of
billed services behind that sentence — it's an open-source Python
library, the same shape as LangGraph, CrewAI, or Strands: `pip install`,
write Python, own the process it runs in. The team behind it is the same
one behind Pydantic itself, and that lineage is the actual pitch: take the
validation library most of the Python web ecosystem already leans on and
point it at the specific place agent code tends to get sloppy — the
boundary between "the model said something" and "my code can trust that
something."

## The core loop: Agent, deps, output

An `Agent` is generic over two types: what it needs to run
(`deps_type`) and what it hands back (`output_type`). Dependencies —
a database connection, an HTTP client, a customer ID — get threaded
through every tool and instruction via `RunContext`, instead of living in
closures or global state:

```python
from dataclasses import dataclass
from pydantic import BaseModel, Field
from pydantic_ai import Agent, RunContext

@dataclass
class SupportDeps:
    customer_id: int
    db: DatabaseConn

class SupportOutput(BaseModel):
    advice: str = Field(description="Advice returned to the customer")
    block_card: bool
    risk: int = Field(ge=0, le=10)

support_agent = Agent(
    "openai:gpt-5.2",
    deps_type=SupportDeps,
    output_type=SupportOutput,
    instructions="Give the customer support and judge the risk of their query.",
)

@support_agent.tool
async def customer_balance(ctx: RunContext[SupportDeps], include_pending: bool) -> float:
    """Returns the customer's current account balance."""
    return await ctx.deps.db.customer_balance(ctx.deps.customer_id, include_pending)

result = await support_agent.run("What is my balance?", deps=SupportDeps(123, db))
result.output  # a SupportOutput instance, not a dict
```

The tool's docstring becomes the description the model sees; its
parameters, minus `ctx`, become the tool's input schema — Pydantic
generates both from the function signature rather than a separate schema
you maintain by hand. A wrong `deps_type` or a tool that returns the wrong
shape is a type-checker error at write time, not something that only
surfaces when a particular code path runs in production.

## Structured output is enforced, not requested

Passing a Pydantic model as `output_type` doesn't just ask the model
nicely for JSON — a validation failure feeds straight back into the same
loop a bad tool call would use. Raise `ModelRetry` from inside a tool, or
from an `@agent.output_validator`, and the model sees the failure message
and gets another turn to fix it, rather than the run failing outright:

```python
from pydantic_ai import Agent, ModelRetry, RunContext

@agent.output_validator
async def validate_sql(ctx: RunContext[DatabaseConn], output: Output) -> Output:
    try:
        await ctx.deps.execute(f"EXPLAIN {output.sql_query}")
    except QueryError as e:
        raise ModelRetry(f"Invalid query: {e}") from e
    return output
```

That validator runs arbitrary code — here, an actual `EXPLAIN` against a
real database — so "the output validates" can mean "the schema shape is
right" or "this SQL is actually runnable against this schema," and the
model gets to correct either kind of failure the same way.

## Multi-agent delegation is just a function call

There's no separate orchestrator object for multi-agent setups. A
delegate agent is called from inside a tool, the same as any other async
call, and its output is returned like any other tool result:

```python
@joke_selection_agent.tool
async def joke_factory(ctx: RunContext, count: int) -> list[str]:
    r = await joke_generation_agent.run(f"Please generate {count} jokes.", usage=ctx.usage)
    return r.output
```

The one thing worth wiring through deliberately is `usage=ctx.usage` —
without it, token and request counts from the delegate agent's own calls
to the model don't show up in the parent's usage total, which matters the
moment you're enforcing a `UsageLimits` cap across a chain of agents
calling agents.

## Tools beyond your own functions: MCP as a client

`MCPToolset` puts a remote MCP server's tools into the exact same
toolset list as `@agent.tool` functions — the model can't tell the
difference between a tool backed by a local Python function and one
proxied to a server over stdio or streamable HTTP:

```python
from pydantic_ai import Agent
from pydantic_ai.mcp import MCPToolset

toolset = MCPToolset("http://localhost:8000/mcp")
agent = Agent("openai:gpt-5.2", toolsets=[toolset])
```

This is the client side only — Pydantic AI reaches out to MCP servers
someone else runs; it doesn't stand one up for you the way AgentCore's
Gateway turns an OpenAPI spec into one.

## Getting out of the local process: durable execution

The framework's own answer to "what if the process dies mid-run" isn't a
managed runtime — it's a wrapper around an existing `Agent` that hands
each step to a workflow engine you operate yourself. `TemporalAgent`
registers the agent's model calls and tool calls as Temporal activities;
`DBOSAgent` does the equivalent against a Postgres-backed durability
model. Either way the agent code itself doesn't change, only how its
steps get checkpointed:

```python
from pydantic_ai import Agent
from pydantic_ai.durable_exec.temporal import PydanticAIPlugin, TemporalAgent

agent = Agent("openai:gpt-5.2", instructions="You're an expert in geography.")
temporal_agent = TemporalAgent(agent)
# steps now resume from the last completed activity on a crash, instead of restarting
```

Compare this to AgentCore's Runtime, which gets session persistence and
crash isolation from a managed microVM you never provision. Here the
resumability is real, but the Temporal cluster or the DBOS-configured
Postgres instance backing it is infrastructure you stand up and run, not
a service AWS bills you for by the second.

## Where this sits relative to the platform frameworks

The [AWS AgentCore](../aws-agentcore/index.md) chapter names pydantic.ai
as the wrong comparison for Harness, and the reasoning holds from this
side too: Harness is config in, AWS operates the VM and the loop; Pydantic
AI is code you write and deploy wherever you already run Python. Next to
LangGraph's explicit graph-and-state-machine model or CrewAI's
role-and-crew abstraction, Pydantic AI's distinguishing bet is narrower —
generics propagated through `Agent[DepsT, OutputT]` so a static type
checker catches a mismatched dependency or a wrong output type before the
code runs at all, and "another agent" as a thing you `await` from a tool
rather than a node you wire into a separate graph object. That's less
machinery to learn than an explicit state graph, and less visual — there's
no diagram of the control flow to look at, just the call graph a normal
Python debugger already understands.

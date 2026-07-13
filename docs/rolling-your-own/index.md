---
icon: lucide/hammer
---

# Frameworks: Rolling Your Own

Every framework in this book — [Pydantic AI](../pydantic-ai/index.md),
[Strands Agents](../strands-agents/index.md), the harness underneath
[AgentCore](../aws-agentcore/index.md) — is a wrapper around the same
few dozen lines of Python. A message list, a call to the model, a check
of whether it asked for a tool, a local function call, and a loop back
around. None of that is proprietary or hidden; it's the reference
implementation printed on the front page of every provider's tool-use
docs. This chapter builds that loop directly against the Anthropic API,
with no framework in between, to show exactly what the decorators and
config objects in the other chapters are standing in for — and where
that trade stops being a good one.

## The whole loop is a while loop

An agent is a conversation where some replies are followed by a local
function call instead of a reply to the user. That's the entire
mechanism:

```python
from anthropic import Anthropic

client = Anthropic()

TOOLS = [
    {
        "name": "get_weather",
        "description": "Get the current weather for a city.",
        "input_schema": {
            "type": "object",
            "properties": {"city": {"type": "string"}},
            "required": ["city"],
        },
    }
]

def get_weather(city: str) -> str:
    return f"It's sunny in {city}."

DISPATCH = {"get_weather": get_weather}

def run_agent(prompt: str, system: str = "") -> str:
    messages = [{"role": "user", "content": prompt}]
    while True:
        response = client.messages.create(
            model="claude-sonnet-5",
            max_tokens=1024,
            system=system,
            tools=TOOLS,
            messages=messages,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return "".join(b.text for b in response.content if b.type == "text")

        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = DISPATCH[block.name](**block.input)
                results.append(
                    {"type": "tool_result", "tool_use_id": block.id, "content": output}
                )
        messages.append({"role": "user", "content": results})
```

`run_agent("What's the weather in Turku?")` makes one call, sees
`stop_reason == "tool_use"`, runs `get_weather(city="Turku")` locally,
feeds the string back as a `tool_result`, and loops — the second call
gets a plain text reply and the function returns. Nothing here is
Anthropic-specific in shape; OpenAI's `tool_calls` and Gemini's
`function_call` parts slot into the same loop with different field
names — see [The OpenAI API](../openai-api/index.md) for how widely that
particular shape got copied. What Strands's `@tool` decorator and
Pydantic AI's `@agent.tool`
actually do is generate the `input_schema` dict from a function
signature and route `block.name` to the right Python callable instead of
a hand-written `DISPATCH` table — real conveniences, but conveniences
over this loop, not a different loop.

## Tool dispatch is a dict, not a decorator

`DISPATCH` above is doing, by hand, what every framework's tool registry
does under a nicer name: mapping a string the model produced to a
function your process actually runs. With one tool that dict looks
redundant. With ten tools, each needing its own schema kept in sync with
its own signature, the redundancy is exactly what a decorator earns its
keep removing — `inspect.signature()` plus a docstring gets you most of
the way to `input_schema` without maintaining it twice:

```python
import inspect
from typing import get_type_hints

def tool(fn):
    hints = get_type_hints(fn)
    props = {name: {"type": "string"} for name in hints if name != "return"}
    fn.schema = {
        "name": fn.__name__,
        "description": (fn.__doc__ or "").strip(),
        "input_schema": {"type": "object", "properties": props, "required": list(props)},
    }
    return fn

@tool
def get_weather(city: str) -> str:
    """Get the current weather for a city."""
    return f"It's sunny in {city}."
```

That's most of a minimal `@strands.tool` in about ten lines — enough to
see that the decorator isn't magic, and enough to see why nobody stops
here. A real version needs to turn `int`, `list[str]`, and nested
Pydantic models into correct JSON Schema, which is exactly the part
worth not maintaining by hand once there's more than one type in play.

## History is a list you own

`messages` in `run_agent` above is not a detail — it's the same fact the
[Pydantic AI chapter](../pydantic-ai/index.md) makes about
`message_history`, just with the wrapper removed so there's nothing left
to obscure it. There is no session object, no thread, nothing server-side
tracking that this is a continuation. The list you pass on the next call
*is* the conversation; skip appending to it and the model has no idea a
previous turn happened. A second call to `run_agent` with a fresh
`messages = [...]` is indistinguishable, from the model's side, from a
different user entirely.

Persisting a conversation across two HTTP requests, a page reload, or a
Slack thread's next message means serializing that same list — to a row
in Postgres, a Redis key, a session cookie's payload — and loading it
back before the next call. That's the whole of "conversation state" once
a framework isn't standing between you and it: a list of dicts, JSON-safe
except for the raw `response.content` blocks, which need
`.model_dump()` (or `json.dumps(default=lambda b: b.model_dump())`) to
serialize cleanly.

## Structured output: validate, don't trust

`output_type` in Pydantic AI is a Pydantic model plus a retry loop
already wired up. Without the framework, both pieces are visible
separately — parse, and if parsing fails, feed the error back in instead
of raising:

```python
from pydantic import BaseModel, ValidationError

class Risk(BaseModel):
    level: int
    reason: str

def run_structured(prompt: str, model: type[BaseModel], attempts: int = 3):
    messages = [{"role": "user", "content": prompt}]
    for _ in range(attempts):
        response = client.messages.create(
            model="claude-sonnet-5", max_tokens=1024, messages=messages,
        )
        text = "".join(b.text for b in response.content if b.type == "text")
        try:
            return model.model_validate_json(text)
        except ValidationError as e:
            messages.append({"role": "assistant", "content": text})
            messages.append({"role": "user", "content": f"That didn't validate: {e}. Try again, output only JSON."})
    raise ValueError("model never produced valid output")
```

This is a strictly worse version of what `ModelRetry` gives you — no
schema passed to the model up front (so more first-attempt failures), no
integration with the tool-calling loop (a validator can't also be a
tool). It's here to make the point that "structured output" is not a
capability the model has and a plain API call lacks; it's a validate-and-
retry loop that a framework will happily write for you, but that you can
also write in nine lines when the framework isn't the right shape for
the rest of the problem.

## Pausing for approval: same primitive, no framework

The [Pydantic AI chapter](../pydantic-ai/index.md#pausing-without-a-workflow-engine-deferred-tools)
covers `ApprovalRequired` and `DeferredToolRequests` as framework
features. Underneath, "pause and wait for a human" is just the loop
above exiting early instead of dispatching a tool call:

```python
NEEDS_APPROVAL = {"delete_file"}

def run_agent_with_approval(messages: list):
    response = client.messages.create(
        model="claude-sonnet-5", max_tokens=1024, tools=TOOLS, messages=messages,
    )
    messages.append({"role": "assistant", "content": response.content})

    pending = [b for b in response.content if b.type == "tool_use" and b.name in NEEDS_APPROVAL]
    if pending:
        return {"status": "waiting_for_approval", "messages": messages, "pending": pending}
    # ... dispatch the rest as usual, or return the final text
```

Resuming is appending one `tool_result` per pending call — `"approved"`
or `"denied: <reason>"` as the content — and calling the loop again with
the same `messages`. Nothing about this needs a workflow engine or a
managed runtime; it needs the caller to hold onto `messages` and
`pending` for however long the human takes, the same storage problem as
any other conversation history. What a real implementation adds is
exactly what you'd expect by now: matching approvals back to the right
`tool_use_id` when several are pending at once, and a timeout for the
approval that never comes.

## Sub-agents are just recursive calls

Pydantic AI's multi-agent delegation is "call another agent from inside
a tool." With no framework, that's calling `run_agent` from inside a
function that's itself in `DISPATCH`:

```python
def delegate_to_researcher(question: str) -> str:
    return run_agent(question, system="You are a research assistant. Be terse.")

DISPATCH["ask_researcher"] = delegate_to_researcher
```

A "multi-agent system" here is not a new concept — it's the same
message-list-and-loop function, called recursively, with a different
`system` string and a different `TOOLS` list each time. The interesting
engineering problem this glosses over — propagating token usage from the
inner call up to a cap enforced on the outer one, which Pydantic AI
handles via `usage=ctx.usage` — doesn't go away by rolling your own; it
just becomes a plain function argument you have to remember to thread
through yourself, with no framework to complain when you forget.

## What you give up by rolling your own

None of the above is an argument that frameworks are unnecessary — it's
an argument that they're *legible*: every one of them is doing something
you could write yourself, which is why it's possible to show the
underlying version in a page each. The reasons to reach for Pydantic AI,
Strands, or a managed runtime instead of the loop above are specific, not
vague:

- **Schema generation at scale.** One tool, hand-write the dict. Twenty
  tools with nested types across a team, and a decorator that derives
  `input_schema` from a signature stops being sugar and starts being the
  thing preventing schema drift.
- **Crash recovery.** The loop above dies with the process. `TemporalAgent`
  and `DBOSAgent` checkpoint every step server-side so a crash mid-run
  resumes on its own — real infrastructure, not a framework trick, but
  infrastructure you'd have to stand up yourself either way.
- **MCP client plumbing.** Connecting to a remote MCP server, keeping its
  tool list in sync with your local ones, and handling reconnects is
  maintainable by hand for one server and genuinely tedious for several.
- **Streaming, parallel tool calls, retries with backoff.** All three fit
  inside the same loop shape but add real edge-case handling — a
  framework has usually already hit the edge case you haven't yet.

A single tool, a single turn shape, full control over retry and error
behavior, no dependency to track through version bumps — that's the
territory where the loop above is not a toy but the actual right answer.
The moment the honest list above starts including two or three items,
that's the signal to stop maintaining the loop by hand and reach for one
of the frameworks in the rest of this book.

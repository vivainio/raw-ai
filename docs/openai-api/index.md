---
icon: lucide/plug
---

# The OpenAI API

OpenAI didn't invent the chat-shaped request — system message, then a flat
list of user/assistant turns — and didn't invent function calling either.
What it has is the version everyone else's server now knows how to speak.
When a self-hosted inference server, a routing service, or a local model
runner advertises itself as "OpenAI-compatible," there's one specific thing
being promised: it accepts a POST to something shaped like
`/v1/chat/completions`, with a JSON body carrying `model` and `messages`,
and answers with a JSON body carrying `choices[0].message`. Ollama, vLLM,
llama.cpp's server mode, Groq, Together, Fireworks, LM Studio, and Azure's
own hosting of OpenAI's models all expose that endpoint, whether or not the
model actually running behind it came from OpenAI. Proxy layers like
LiteLLM go further and use this shape as their own internal
lingua franca, translating everything else — including Anthropic's Messages
API, which isn't shaped like this at all — into it and back out again.
That's the actual meaning of "standard" here: not a spec body, not a
committee, just the first API that shipped with function calling attached
and got copied widely enough that copying it became the path of least
resistance for the next fifteen inference servers.

## The part that got cloned

A minimal Chat Completions request and response:

```python
from openai import OpenAI
client = OpenAI()

completion = client.chat.completions.create(
    model="gpt-5.6",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Hello!"},
    ],
)
completion.choices[0].message.content
```

Swap the base URL to a local Ollama instance or a Groq endpoint, keep the
key and the shape identical, and most client code doesn't notice the
difference. That's deliberate on the cloners' side — matching this one
endpoint is the cheapest way to be a drop-in replacement for every app,
script, and framework default already written against the `openai` Python
or Node SDK.

Function calling followed the same path. A tool is declared as a name, a
description, and a JSON Schema for its parameters; a model that wants to
call one returns a `tool_calls` array on the assistant message instead of
plain text:

```json
{
  "role": "assistant",
  "tool_calls": [
    {
      "id": "call_FthC9qRpsL5kBpwwyw6c7j4k",
      "type": "function",
      "function": {
        "name": "get_current_temperature",
        "arguments": "{\"location\": \"San Francisco, CA\", \"unit\": \"Fahrenheit\"}"
      }
    }
  ]
}
```

The result goes back as its own message, `role: "tool"`, carrying the
matching `tool_call_id`. Compare that to the [Agent Dialogue chapter's
tool_use/tool_result
pair](../agent-dialogue/index.md#calling-a-tool), pulled from an actual
Claude Code session: Anthropic's `input` on a `tool_use` block is a real
parsed object, and the result comes back as a `tool_result` content block
nested inside a `user`-role message rather than a sibling `tool` role.
Both encode the identical idea — a predicted call, an ID to correlate it,
a result fed back on the next turn — but neither the role structure nor
the argument encoding lines up. `arguments` being a JSON string that the
caller has to parse a second time, instead of a value already sitting in
the object tree, is a specific and slightly awkward choice that stuck
because OpenAI's shape was first, not because it's the shape anyone would
pick from scratch. Providers cloning Chat Completions kept it anyway —
the whole point of "OpenAI-compatible" is not introducing a difference
where a client wouldn't expect one.

## Where compatibility runs out

Chat Completions has no memory of its own — every call resends the full
`messages` array, the same statelessness [Pydantic AI's `agent.run()`
starts from](../pydantic-ai/index.md#theres-no-session-object). That's the
other reason it was easy to clone: a stateless endpoint has no session
store for a compatible server to also have to replicate. OpenAI's own
newer surface, the Responses API, gives that up in favor of a variant of
statefulness that's easy for a compatible clone to skip entirely:

```python
response = client.responses.create(model="gpt-5.6", input="tell me a joke")
second = client.responses.create(
    model="gpt-5.6",
    previous_response_id=response.id,
    input=[{"role": "user", "content": "explain why this is funny."}],
)
```

`previous_response_id` chains onto a response OpenAI already stored
server-side, closer to [AgentCore Memory auto-persisting a session by
ID](../aws-agentcore/index.md) or [Strands'
`SessionManager`](../strands-agents/index.md#statefulness-is-a-constructor-argument-not-something-you-build)
than to anything a caller assembles itself — except scoped to a chain of
responses rather than a durable thread object, and only usable against
OpenAI's own servers, since it depends on state OpenAI is holding, not
state the request carries. A self-hosted vLLM box answering `/v1/chat/completions`
has no way to also be the thing `previous_response_id` points at.

The request shape diverges too. Responses separates `instructions` from
`input` instead of folding a system message into the same array, reads
generated text from `output_text` instead of `choices[0].message.content`,
and defines Structured Outputs one level deeper — `text.format` instead of
`response_format.json_schema`:

```python
response = client.responses.create(
    model="gpt-5.6",
    input="Jane, 54 years old",
    text={
        "format": {
            "type": "json_schema",
            "name": "person",
            "strict": True,
            "schema": {
                "type": "object",
                "properties": {
                    "name": {"type": "string"},
                    "age": {"type": "number"},
                },
                "required": ["name", "age"],
                "additionalProperties": False,
            },
        }
    },
)
```

None of that is something a Chat-Completions-compatible clone gets for
free. "OpenAI-compatible" turned out to mean "compatible with Chat
Completions specifically" — the surface that was easy to clone became the
de facto standard by default, and OpenAI's own newer, statefully-designed
API isn't part of what anyone else bothered to copy.

## Tools OpenAI runs, not tools you write

Responses also ships hosted tools — `web_search`, `code_interpreter`, and
others — that the model can call without the caller implementing anything:

```python
answer = client.responses.create(
    model="gpt-5.6",
    input="Who is the current president of France?",
    tools=[{"type": "web_search"}],
)
```

This is the point past which "compatible" stops being possible even in
principle, not just in practice. A user-defined tool is a name and a JSON
Schema — any server can accept that shape and hand back a `tool_calls`
entry, because running the tool was always the caller's job anyway. A
hosted tool needs the actual infrastructure behind `web_search` — crawling,
an index, a rendering pipeline — sitting behind the API. Cloning the
request shape buys nothing if there's nothing on the other end to execute
it against.

## Chat Completions' one real cost: reasoning gets thrown away

For a reasoning model, Chat Completions' statelessness isn't just an
inconvenience for session management — OpenAI's own guidance says it
degrades results. The reasoning a model does before producing a tool call
isn't part of the `messages` array, so a Chat Completions round-trip
resends the conversation without it; the model re-derives whatever
reasoning it needs on the next turn instead of continuing from the last
one. In a multi-step agentic tool-calling loop this shows up as worse
accuracy and higher reasoning-token spend per turn than the same loop run
through Responses, where reasoning items persist as part of the stored
response chain. OpenAI's guidance is explicit that this only bites in
multi-step agentic use — a single-turn Q&A call doesn't accumulate enough
reasoning for the gap to matter.

## OpenAI's own churn

The Assistants API — OpenAI's first attempt at a stateful, tool-using
agent abstraction, with its own threads and runs — was deprecated on
2025-08-26 and is scheduled for removal from the API entirely on
2026-08-26, a month from now as this is written. Its replacement is the
Responses API plus a separate Conversations API for persistent threads.
Worth sitting with for a moment: the company that set the de facto
standard for the *stateless* tool-calling shape has already shipped and
retired one stateful agent API of its own, and is one migration into a
second. The Chat Completions shape survives everywhere because it's
simple enough that cloning it was nearly free; nothing OpenAI has shipped
since has been simple in that same way, which is a reasonable guess at
why none of it got cloned the same way either.

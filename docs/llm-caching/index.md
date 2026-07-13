---
icon: lucide/database-zap
---

# Prompt Caching

Every message you send an LLM carries the whole conversation with it — the
API has no memory between requests, so the system prompt, the tool
definitions, and every prior turn get re-read and re-priced on every single
call. For a short chat that's fine. For an agent with a large system prompt,
a dozen tool schemas, and a transcript that grows every turn, it means
paying full price to re-process the same few thousand tokens hundreds of
times in a row. Prompt caching is the fix: mark a point in the prompt, and
the API reuses what it already processed up to that point instead of
redoing the work.

## The invariant: it's a prefix match

A cache entry is keyed on the exact bytes of the prompt up to a marked
point. Change one byte anywhere before that point — a timestamp, a
reordered JSON key, a different tool in the list — and the entry no longer
matches. Everything from that byte onward reprocesses at full price, even
if the rest of the prompt is identical to a hundred prior requests.

That single rule explains most of what follows. The API builds a request in
a fixed order — tools, then system prompt, then messages — so a cache
marker placed on the last system block covers tools and system together;
one placed on the last message in a conversation covers everything before
it. Whatever comes after the last marker is never cached, by design — that's
where the part of the prompt that actually changes every turn belongs.

## Placing a breakpoint

The marker is a `cache_control` block attached to a content block:

```python
response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    system=[{
        "type": "text",
        "text": LONG_SYSTEM_PROMPT,
        "cache_control": {"type": "ephemeral"},
    }],
    messages=[{"role": "user", "content": "Summarize the key points"}],
)
```

For a growing multi-turn conversation, the marker moves to the last block
of the most recently appended turn each time — the API walks backward
looking for the nearest prior cache entry, so earlier breakpoints stay
valid and hits accumulate as the conversation gets longer. There's also a
top-level `cache_control` on the request that auto-places the marker on the
last cacheable block, which is the simplest option when fine-grained
placement isn't worth the code.

A request allows up to four separate breakpoints — useful when a prompt has
several stability tiers (a frozen system prompt, a per-session block of
retrieved context, then the live conversation).

## Reads are cheap, writes aren't free

A cache entry isn't free to create. Writing one costs a premium over a
normal input token — more for a longer-lived entry — while reading a hit
back costs a small fraction of the normal rate. That shapes when caching is
worth it: a single request that's never repeated pays the write premium for
nothing, since there's no second read to recoup it. It only pays off across
multiple requests that share the same prefix, and a longer-lived cache
needs more reuse to break even than a short-lived one, because the write
costs more up front.

Two lifetimes are available: a default that lasts a few minutes, and a
longer one for traffic with bigger gaps between requests. Pick the short
one for anything conversational, where the next turn is seconds away, and
the long one for bursty patterns — a batch job that fires every so often
against the same shared context, say — where the requests are real but
spaced out enough that the short lifetime would keep expiring between them.

## There's a minimum size

A `cache_control` marker on a short prompt doesn't error — it just silently
doesn't cache. Each model has its own minimum token count below which a
marked prefix is too small to bother writing. It's easy to miss: the
request succeeds, the response looks normal, and the only symptom is that
the usage numbers never show a cache hit. Which is the actual way to tell
whether caching is working at all — not by reading the code, but by reading
the response.

## Checking that it's actually happening

The response carries three numbers worth watching: tokens written fresh to
cache, tokens served from a prior cache entry, and tokens that weren't
cached at all. If the middle number stays at zero across a run of
requests that should share a prefix, something upstream is invalidating the
cache on every call, and the fix is to find that thing rather than to
re-check the `cache_control` placement, which is probably fine.

## What breaks it

The prefix-match rule means anything that changes the rendered bytes before
a marker breaks the cache at and after that point. In practice, the usual
culprits are the same everywhere:

- **A timestamp or request ID baked into the system prompt.** `"Current
  time: 14:32:07"` at the top of the prompt means no two requests ever
  share a prefix.
- **Non-deterministic serialization.** Dumping a dict without sorting keys,
  or iterating a set, can silently reorder the same logical content into
  different bytes from one call to the next.
- **A tool set that varies per request or per user.** Tools render first,
  before the system prompt — adding, removing, or reordering even one tool
  invalidates everything downstream of it.
- **Switching models mid-conversation.** Caches are scoped per model; a
  different model ID is a different cache, full stop.

Some of these invalidate more than others. The API actually has separate
cache tiers for tools, system, and messages — a change to `tool_choice` or
to whether images are present invalidates the system and message tiers but
leaves the tools tier alone; a change to the tool list itself invalidates
all three. So not every request-shape change is equally expensive, but the
one thing that's always expensive is touching content that sits early in
the render order.

## Extended thinking and tool-heavy loops

Two things worth knowing if the agent uses thinking or runs long tool-use
loops:

**Thinking blocks participate in the same prefix rule** — turning thinking
on or off mid-conversation invalidates system and message caching the same
way a `tool_choice` change does, so it's not something to toggle turn to
turn if caching matters.

**Long tool-use turns have a lookback limit.** A breakpoint only looks back
twenty content blocks to find its prior entry. An agentic loop that racks
up more than twenty `tool_use`/`tool_result` pairs in a single turn can walk
right past that window, and the next request's marker silently misses a
cache entry that's still technically there — just further back than the
API will look. The fix is the same one used for long system prompts:
place an intermediate breakpoint every so often within the turn instead of
only at the end.

## The mid-conversation problem

The prefix rule creates one recurring conflict: agents often need to inject
something dynamic — today's date, a mode switch, a fact learned mid-run —
and the obvious place to put it is the system prompt, which is exactly the
part of the request that sits earliest and is most expensive to
invalidate. The usual fix is to stop treating the system prompt as mutable
at all: keep it frozen, and put anything that changes into the message
list instead, appended after the last cache marker rather than edited into
the prefix. A one-line fact added at turn twelve costs nothing upstream of
it; the same fact edited into the system prompt invalidates every turn that
came before.

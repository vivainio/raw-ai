---
icon: lucide/messages-square
---

# Agent Dialogue

A coding agent looks like a chat window. Underneath, there's no chat — there's
a list. Every request to the model resends the entire conversation so far:
system prompt, then a flat sequence of messages, each one tagged `user` or
`assistant` and holding one or more typed content blocks. The model has no
memory between requests; the illusion of an ongoing conversation is just the
harness (Claude Code, in this case) replaying the same growing list every
time and appending whatever comes back.

That list is also, literally, a file. Claude Code writes one JSON object per
line to `~/.claude/projects/<project>/<session-id>.jsonl` as the session
runs. The examples below aren't invented — they're pulled straight out of
this book's own working session, the one where this chapter got written.

## The system prompt

The system prompt is the one message the model doesn't see coming from you.
It's assembled by the harness before your first word and doesn't change
mid-session the way a normal message would. Broadly it carries a few kinds
of thing, stacked together:

- **Identity and ground rules** — what kind of assistant this is, how to
  handle destructive actions, when to ask before proceeding.
- **Environment context** — facts the model can't otherwise know: working
  directory, platform, today's date, git branch and status. Something like:

  ```
  Working directory: /home/v/r/raw-ai
  Platform: linux
  Today's date: 2026-07-13
  Git branch: main (clean)
  ```

- **Tool definitions** — a name, description, and JSON schema for every tool
  the model is allowed to call this session (`Bash`, `Read`, `Edit`, and so
  on). This is the part that scales with how much you've bolted on: every
  configured MCP server, every skill description, adds more of it, whether
  or not the conversation ends up using it.
- **User-supplied instructions** — `CLAUDE.md` files, both global and
  per-project, get folded in here too. From the model's point of view
  there's no line between "instructions the product author wrote" and
  "instructions you wrote in a file" — both just arrive as more system text.

None of this is negotiated with the model. It's context, not conversation —
the first fact the model is handed, not something it can talk back to.

## Everything after that is just messages

Once the system prompt is set, the rest of the session is a plain array.
Trimmed down, a single exchange looks like this:

```json
[
  { "role": "user", "content": "list the files here" },
  {
    "role": "assistant",
    "content": [
      { "type": "text", "text": "I'll check the directory." },
      { "type": "tool_use", "id": "toolu_01ABC", "name": "Bash",
        "input": { "command": "ls -la" } }
    ]
  },
  {
    "role": "user",
    "content": [
      { "type": "tool_result", "tool_use_id": "toolu_01ABC",
        "content": "README.md\ndocs/\nzensical.toml" }
    ]
  }
]
```

Text you typed, text the model wrote, a tool it decided to run, and the
result of that tool are all just entries in the same list, distinguished
only by `role` and `type`. There's no separate channel for "the model is
doing something" versus "the model is talking to you" — it's all one
sequence, replayed in full on every turn.

## Calling a tool

"The model calls a tool" is a convenient way to talk about it, but the
mechanism is plainer than that. The model predicts a JSON object — a
`tool_use` block with a name and an input matching that tool's schema — and
stops. It doesn't run anything. The harness reads that block, actually
executes the command, and appends the result as a `tool_result` block,
tagged with the same id, in the next message. Only then does the model see
what happened and continue.

Here's a real pair from this session — the model wanted to look at the repo
before doing anything else:

```json
{
  "type": "tool_use",
  "id": "toolu_01WtMNM44PEjPDp1w2mrQW1X",
  "name": "Bash",
  "input": {
    "command": "ls -la /home/v/r/xml-avant-garde/ && echo \"---\" && find /home/v/r/xml-avant-garde/.github -type f",
    "description": "List xml-avant-garde repo structure and github actions"
  }
}
```

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01WtMNM44PEjPDp1w2mrQW1X",
  "content": "755  .cache/\n755  .claude/\n755  .git/\n755  .github/\n755  docs/\n755  site/\n644  .gitignore  30B\n644  README.md  1.2K\n644  zensical.toml  8.9K\n---\n1F 1D:\n\nworkflows/ docs.yml",
  "is_error": false
}
```

That's the entire protocol for a tool call. No RPC, no persistent
connection to the shell — one predicted JSON block out, one JSON block with
the real-world result back in. Whether that result is a directory listing,
a file's contents, or a web search result makes no structural difference to
the model; it's all text in a `tool_result`.

## The user is also a tool

`AskUserQuestion` — the mechanism the model uses to ask *you* something —
isn't a special protocol feature. It's a tool like any other, with a
schema (questions, options, headers), and the same `tool_use` /
`tool_result` shape as `Bash`. The only thing unusual about it is what the
harness does when it sees that name: instead of running a command, it
renders a UI, blocks, and waits for a human.

Early in this session, before any files existed, the model asked what the
book was actually about:

```json
{
  "type": "tool_use",
  "name": "AskUserQuestion",
  "input": {
    "questions": [{
      "question": "What's the tagline/description for Raw AI, and roughly what topics will it cover?",
      "header": "Book scope",
      "options": [
        { "label": "Practical LLM/agent engineering", "description": "..." },
        { "label": "AI fundamentals from scratch", "description": "..." }
      ]
    }]
  }
}
```

The answer came back exactly the way a shell command's output would — as a
`tool_result` on the next `user`-role message:

```json
{
  "type": "tool_result",
  "content": "Your questions have been answered: \"What's the tagline/description for Raw AI...\"=\"I'll write it myself, just stub it\". You can now continue with these answers in mind."
}
```

Same thing happened later, before the first commit:

```json
{
  "type": "tool_use",
  "name": "AskUserQuestion",
  "input": {
    "questions": [{
      "question": "Ready to commit and push the initial scaffold to origin/main?",
      "header": "Commit & push",
      "options": [
        { "label": "Yes, commit and push", "description": "..." },
        { "label": "Just commit locally", "description": "..." },
        { "label": "Leave unstaged", "description": "..." }
      ]
    }]
  }
}
```
```
"Yes, commit and push"
```

From the model's side, there is no structural difference between a human
clicking a button, a human typing an answer, and a shell command printing
to stdout. All three arrive as the same shape: a `tool_result` block
answering a `tool_use` block. The chat bubble where you type your reply is
a UI convenience — under the hood, an ordinary typed message from you and
the answer to an `AskUserQuestion` call both just become the next entry in
the list.

## Saying no

Permission prompts work the same way. When a tool call needs approval and
you decline it, the harness doesn't drop the request — it still writes a
`tool_result`, just one that says the call failed and why:

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_019CFbKvhmkwdjtUwg6i7F2Q",
  "is_error": true,
  "content": "The user doesn't want to proceed with this tool use. The tool use was rejected (eg. if it was a file edit, the new_string was NOT written to the file). STOP what you are doing and wait for the user to tell you how to proceed.\n\nNote: The user's next message may contain a correction or preference. Pay close attention — if they explain what went wrong or how they'd prefer you to work, consider saving that to memory for future sessions."
}
```

That's from this same session — a `git status` check that got interrupted
mid-flight. The model doesn't receive a system-level "denied" signal
outside the conversation; it receives a `tool_result` like any other,
worded to make the refusal unambiguous and to steer it toward stopping
rather than retrying. Permission handling isn't a separate mechanism
bolted onto the dialogue — it's the exact same `tool_use`/`tool_result`
loop, with the harness controlling what gets written into the result.

## Not everything labeled "user" is you

Once you've seen enough of a raw session log, one thing stands out: plenty
of `role: user` messages were never typed by a human. Slash commands,
queued messages sent while the model was mid-turn, reminders about newly
available tools, snapshots of file state — all of it gets threaded in as
`user`-role content, because that's the only role the API defines for
"something other than the model." The message you're reading this
paragraph in response to is itself an example: a `<system-reminder>` tag
injected into a `user` message, not something a person wrote by hand.

That's really the whole shape of it. A session is one flat, replayed list.
Personality and rules live in one message at the top. Everything after —
your questions, the model's replies, the commands it runs, what those
commands print, the answers you give when it asks, the refusals when you
say no — is encoded the same way: a role, and one or more typed content
blocks. No hidden channel carries any of it.

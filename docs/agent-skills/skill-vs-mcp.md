# Skill + CLI instead of MCP

MCP wraps each operation as a typed tool: `jira_get_issue`, `jira_search`,
`jira_create_issue`, one schema per call. Those schemas sit in context for the
whole session whether or not the conversation ever touches Jira, and the list
only grows as you expose more operations.

The alternative: ship a normal CLI, and a skill that just tells the agent when
and how to reach for it. The agent already knows how to run a shell command
and read its output — you're not teaching it a new protocol, just pointing it
at a tool.

## Case study: zaira

[zaira](https://github.com/vivainio/zaira) is a CLI for Jira and Confluence —
`zaira get FOO-123`, `zaira search "login bug"`, `zaira create ticket.md`,
`zaira transition FOO-123 Done`, `zaira wiki get 123456`, and so on. Plain
subcommands, markdown/JSON in and out, credentials read from the OS keyring.
Nothing here is agent-specific.

What makes it agent-friendly is a companion skill, installed with:

```sh
zaira install-skills    # drops the skill into ~/.claude/skills/
```

The skill itself is small — description and trigger conditions, then which
subcommand to run for which situation:

```markdown
---
name: zaira
description: Jira/Confluence CLI. Use when the user asks about tickets, sprints, or wiki pages.
---

# zaira

Get a ticket: `zaira get PROJ-123`
Search: `zaira search "<terms>" --project PROJ`
Create from a markdown file with YAML front matter: `zaira create ticket.md`
```

There's no schema to maintain, no server process, no separate "agent
interface" alongside the human one. `zaira get FOO-123` is the whole
integration — the agent runs it, you can run the identical command yourself
in your own terminal to see exactly what it saw.

## Why this wins for CLI-shaped tools

- **Cost.** Until the skill's description matches what you're doing, all that
  sits in context is one name and one line of description — not a full tool
  schema per subcommand.
- **Debuggability.** Every command the agent runs is one you can paste and
  rerun yourself. An MCP call is a JSON-RPC exchange with a running server;
  there's no equivalent "just run it in the terminal."
- **No process to manage.** zaira reads credentials straight from the OS
  keyring (Keychain, Secret Service, Credential Manager). There's no MCP
  server to start, authenticate, and keep alive alongside the agent.
- **No idle bloat.** Each configured MCP server is a process — often a `node`
  process, sometimes a whole Docker container — that starts with your editor
  or agent session and sits there whether you use it that day or not. Add a
  server per integration and you accumulate a pile of these, memory and CPU
  spent on tools nobody's calling. A CLI only runs while a command is
  actually executing.
- **One tool, two users.** The CLI a human runs by hand is the exact CLI the
  agent runs. Nothing agent-only to keep working.

MCP still earns its keep for genuinely stateful or streaming protocols, or
when there's no way to expose the operation as a local command at all. For
"call an API, get text back" tools — which is most of them — a CLI plus a
skill that explains it is less to build and less to carry in context.

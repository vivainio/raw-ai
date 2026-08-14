---
icon: lucide/terminal
---

# Claude Code: Commands, Configuration, and Hooks

Claude Code has two command surfaces. The `claude` executable starts or resumes
a process from the shell; commands beginning with `/` operate inside an active
session. Around those commands sit settings files, permission rules, hooks,
skills, subagents, and MCP servers. This chapter is a map of that surface: what
each control changes, where its state is stored, and how the pieces connect.

The command set changes fairly often. `/help` shows what is present in the
installed version, while Anthropic's [commands reference](https://code.claude.com/docs/en/commands)
and [CLI reference](https://code.claude.com/docs/en/cli-reference) describe the
current release. Some commands also depend on platform, account plan, or an
enabled experimental feature.

## Starting from the shell

With no prompt, `claude` opens an interactive session in the current directory:

```sh
claude
```

A positional argument becomes the first prompt:

```sh
claude "explain how authentication reaches the database"
```

`-p` or `--print` runs non-interactively, prints the result, and exits. It is the
form used in pipes and scripts:

```sh
claude -p "summarize the changes under src/"
git diff | claude -p "describe this diff"
```

`--output-format` controls the representation returned by print mode. `text` is
the default, `json` returns one result object, and `stream-json` emits a stream
of events. `--input-format stream-json` accepts a corresponding event stream on
standard input. `--json-schema` asks for a result conforming to a supplied JSON
Schema.

```sh
claude -p --output-format json "list the public modules"
claude -p --json-schema '{"type":"object","properties":{"modules":{"type":"array","items":{"type":"string"}}},"required":["modules"]}' \
  "list the public modules"
```

`-c` or `--continue` reopens the most recent conversation for the current
project. `-r` or `--resume` accepts a session id or name; without a value it
opens a picker.

```sh
claude --continue
claude --resume auth-investigation
claude --resume 550e8400-e29b-41d4-a716-446655440000
```

`--fork-session` combines with `--continue` or `--resume`. It copies the loaded
history into a new session id instead of appending to the original session.

```sh
claude --continue --fork-session
claude --resume auth-investigation --fork-session
```

`--worktree [name]` starts Claude in a Git worktree. `--add-dir path` grants the
session access to an additional directory, and `--cwd path` chooses the initial
working directory. Model and permission state can be selected at launch with
flags such as `--model`, `--effort`, `--permission-mode`, `--allowedTools`, and
`--disallowedTools`.

!!! note "Codex"
    The comparable non-interactive entry point is `codex exec`. Codex reads
    defaults from `~/.codex/config.toml`; Claude Code reads JSON settings from
    the locations described later in this chapter.

## How slash commands are parsed

A slash command is recognized only at the beginning of a submitted message.
Text after its name is passed as arguments:

```text
/compact retain the API decisions and failing test output
```

Typing `/` opens the command menu. A command submitted while Claude is
responding is normally queued until the response finishes. Status-like commands
such as `/status`, `/tasks`, and `/usage` can run immediately. Skills are a
special case: several can be placed at the beginning of one message, followed
by the text passed to them.

```text
/security-check /release-check version 2.4.0
```

## Sessions and conversation copies

Claude Code stores session transcripts locally and associates them with a
project directory. A session has an id and may also have a name.

### `/rename [name]`

`/rename release-debugging` assigns a name to the current session. With no
argument Claude Code generates a name from the conversation. Names can be used
with `/resume` and appear in the session picker.

### `/resume [session]`

`/resume` opens the session picker. `/resume <id-or-name>` loads a particular
conversation. `/continue` is an alias. Resuming retains the session id and adds
new messages to the existing transcript.

### `/clear [name]`

`/clear` starts a new conversation with empty conversational context while
remaining in the project. An optional name labels the conversation being left
behind. `/reset` and `/new` are aliases. The previous conversation remains
available through `/resume`.

### `/branch [name]`

`/branch` copies the conversation through the current message to a new session
id, switches the terminal into that copy, and leaves the original unchanged.
The optional name labels the new session.

```text
/branch try-streaming-parser
```

The command copies conversation history, not the working directory. Both
sessions see whatever files exist in the checkout in which they run. It also
does not execute `git branch`.

### `/fork [prompt]`

`/fork` copies the current conversation into a background session while the
current session remains active. An optional prompt is immediately submitted to
the copy; without one, the copy waits for input in agent view.

```text
/fork trace the legacy authentication path and report the entry points
```

This differs from `/branch` in which copy the terminal continues to control.
`/branch` switches into the copy. `/fork` leaves the terminal in the original
and starts the copy in the background. Current releases instruct a background
copy to create a worktree before editing unless it is configured to edit in
place.

### `/rewind`

`/rewind` opens the checkpoint interface. A selected checkpoint can restore the
conversation, restore code, restore both, or replace a section of the
conversation with a summary. `/checkpoint` and `/undo` are aliases. The exact
file operations are shown before restoration.

### `/btw [question]`

`/btw` asks a side question about the current session without adding the
question and answer to the main conversation history. Running it without a new
question shows the most recent side question and allows earlier answers to be
browsed.

### `/background` and `/exit`

`/background` detaches the entire current session so it can continue as a
background agent; `/bg` is an alias. `claude agents` lists background sessions.
`/exit` terminates an ordinary interactive process, but detaches from an
attached background session without stopping that session. `/quit` is an
alias.

The four similarly named mechanisms therefore change different state:

| Mechanism | Conversation | Files | Git branch |
| --- | --- | --- | --- |
| `/branch` | New foreground session id | Same checkout | Unchanged |
| `/fork` | New background session id | Same checkout or a new worktree | Depends on worktree creation |
| `--fork-session` | New session id at startup | Current checkout | Unchanged |
| `git switch -c name` | Unchanged | Same checkout | New branch |

!!! note "Codex"
    Codex also has `/fork`, but its behavior is defined by the surface in which
    it runs: the desktop command copies a local chat into a new local chat or
    worktree. `/side` opens a temporary side chat. These names are close enough
    that the command menu is a more reliable reference than analogy.

## Goals, plans, and background tasks

### `/goal [condition|clear]`

`/goal` gives the session a persistent completion condition. Claude Code keeps
starting continuation turns until the condition is met. With no argument it
shows the current goal, or the most recently achieved one.

```text
/goal all tests in tests/payments pass and the implementation is committed
```

`/goal clear` removes the active goal. `stop`, `off`, `reset`, `none`, and
`cancel` are accepted as clearing synonyms. A goal differs from an ordinary
prompt because reaching the end of one response does not by itself end the
run.

### `/plan [description]`

`/plan` enters plan mode. An optional argument becomes the task to plan:

```text
/plan replace the file cache with SQLite
```

In plan mode Claude can inspect the project but does not perform ordinary file
edits. Leaving plan mode presents the plan for approval and can include the
permissions the plan expects to require.

### `/tasks`, `/subtask`, and `/batch`

`/tasks` lists background work associated with the session. `/subtask` sends a
bounded piece of work to a subagent whose result returns to the current
conversation. `/batch <instruction>` is a bundled workflow that decomposes a
large change, presents the decomposition, and, after approval, runs units in
separate Git worktrees.

!!! note "Codex"
    Codex exposes `/goal` and `/plan` as well. A Codex goal is also a durable
    objective continued across turns. The storage and continuation controls are
    product-specific even though the command names coincide.

## Context, model, and display

`/context` displays context-window use as a grid broken down by source. `/context
all` expands the item list. `/compact [instructions]` replaces older
conversation content with a summary; the optional text directs what the summary
retains. `/autocompact` displays or changes the threshold at which automatic
compaction occurs.

`/model` opens a model picker or accepts a model name. `/effort` sets the
reasoning-effort level supported by that model. `/fast` toggles the fast service
mode when it is available. `/status` prints session, model, account, and
configuration information. `/usage` shows usage data; `/cost` is its alias.

`/diff` opens an interactive viewer for the current Git diff and for changes
grouped by Claude turn. `/copy [N]` copies a recent assistant response or an
individual code block. `/export [filename]` writes the conversation as plain
text or opens a destination picker.

## Project instructions and working directories

`/init` creates a starter `CLAUDE.md` in the project. `/memory` opens the
interfaces for `CLAUDE.md` files and Claude Code's automatic memory. A project
can contain nested instruction files; Claude Code loads them according to the
files it traverses. User instructions live under `~/.claude/`, while checked-in
project instructions travel with the repository.

`/add-dir <path>` grants file access to another directory for the current
session. `/cd <path>` moves the session's working directory while keeping its
conversation and prompt cache. Adding a directory does not make all of that
directory's `.claude/` configuration active; it primarily changes file access.

!!! note "Codex"
    Codex uses `AGENTS.md` for repository instructions. It searches from the
    configured root toward the working directory and can apply more specific
    files below that point. Its user configuration is rooted at `~/.codex/`.

## Permission rules

`/permissions` opens the permission editor. `/allowed-tools` is an alias. Rules
are grouped into `allow`, `ask`, and `deny`, and may be stored at user, project,
local-project, or managed-policy scope.

```json
{
  "permissions": {
    "allow": [
      "Bash(git status)",
      "Bash(git diff *)"
    ],
    "ask": [
      "Bash(git push *)"
    ],
    "deny": [
      "Read(./.env)",
      "Bash(git reset --hard *)"
    ]
  }
}
```

A rule begins with a tool name and may contain a parenthesized argument
pattern. `Bash(git diff *)` applies to matching shell subcommands;
`Read(./.env)` applies to a file read. MCP tools use their complete generated
tool names. Deny rules are checked before ask rules, and ask rules before allow
rules. A matching allow rule skips the interactive prompt; it does not override
a matching deny or ask rule.

Settings locations determine scope:

| File | Scope | Normally committed |
| --- | --- | --- |
| `~/.claude/settings.json` | All projects for one user | No |
| `.claude/settings.json` | One project | Yes |
| `.claude/settings.local.json` | One project and one user | No |
| Managed settings | Organization policy | Managed externally |

`/sandbox` toggles operating-system sandboxing on supported platforms.
Sandboxing and permission rules are separate: the sandbox limits what a process
can reach, while a permission rule determines whether Claude Code asks, allows,
or rejects a tool invocation.

!!! note "Codex"
    Codex combines approval modes with filesystem and network sandboxing.
    Command rules are written as `prefix_rule()` entries whose decisions are
    `allow`, `prompt`, or `forbidden`. When several prefixes match, the most
    restrictive result wins.

## Hooks

Hooks attach handlers to events in the Claude Code lifecycle. A handler can be
a command, HTTP endpoint, MCP tool, prompt evaluated by a model, or agent-based
check. `/hooks` displays the hooks active in the current session.

The configuration has three levels: event, matcher group, and handler.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "if": "Bash(rm *)",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/block-rm.sh"
          }
        ]
      }
    ]
  }
}
```

`matcher` selects the tool or event subtype. For tool events it receives names
such as `Bash`, `Edit`, `Write`, or `mcp__server__tool`. A plain value is an
exact match, `Edit|Write` selects either name, and values containing regular
expression syntax are treated as JavaScript regular expressions. The optional
handler-level `if` field uses permission-rule syntax to inspect both tool name
and arguments.

### Blocking a shell command

The handler below reads the pending Bash tool call from standard input and
returns a denial for two Git commands:

```sh
#!/usr/bin/env bash

input=$(cat)
command=$(printf '%s' "$input" | jq -r '.tool_input.command // ""')

case "$command" in
  *"git reset --hard"*|*"git clean -fd"*)
    jq -n '{
      hookSpecificOutput: {
        hookEventName: "PreToolUse",
        permissionDecision: "deny",
        permissionDecisionReason: "Command disabled by project hook"
      }
    }'
    ;;
esac
```

For `rm -rf /tmp/build`, the data delivered to a Bash hook contains fields of
this form:

```json
{
  "session_id": "abc123",
  "cwd": "/home/me/project",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "rm -rf /tmp/build"
  }
}
```

The event fires before the tool runs. A matching handler prints its structured
decision to standard output. `permissionDecision: "deny"` cancels the tool call
and returns `permissionDecisionReason` to Claude as the tool error. An exit code
of `2` with the explanation on standard error is the shorter blocking form:

```sh
printf '%s\n' "Command disabled by project hook" >&2
exit 2
```

Exit code `0` with no output makes no decision and passes the call into the
ordinary permission flow. It does not itself approve the call. Other nonzero
exit codes report a hook failure but normally allow processing to continue.

`PreToolUse` can return `allow`, `deny`, `ask`, or `defer` in
`permissionDecision`. It can also return `updatedInput` to replace the tool
arguments and `additionalContext` to add text to Claude's context. If several
hooks make decisions, the order is `deny`, then `defer`, then `ask`, then
`allow`.

### Hook events

The full [hooks reference](https://code.claude.com/docs/en/hooks) defines the
input and permitted output for each event. The main event groups are:

| Event | Point in the lifecycle | Can affect the action |
| --- | --- | --- |
| `SessionStart`, `SessionEnd` | Session opens, resumes, or closes | Context and side effects |
| `UserPromptSubmit` | Before a submitted prompt is processed | Can block the prompt |
| `PreToolUse` | Before a tool call | Can allow, ask, deny, defer, or rewrite |
| `PermissionRequest` | When approval would be requested | Can answer the request |
| `PostToolUse`, `PostToolUseFailure` | After success or failure | Can add feedback; cannot undo side effects |
| `SubagentStart`, `SubagentStop` | Around a subagent run | Context or continuation control |
| `Stop` | Claude is about to finish a turn | Can make the turn continue |
| `PreCompact`, `PostCompact` | Around context compaction | Capture or restore context |
| `WorktreeCreate`, `WorktreeRemove` | Worktree lifecycle | Can supply or clean up a worktree |
| `ConfigChange`, `FileChanged` | Configuration or watched file changes | Notification or blocking varies by event |

The effect of exit code `2` depends on the event. It blocks `PreToolUse`, denies
`PermissionRequest`, and prevents `Stop`; for events that occur after an action,
it can only affect subsequent processing.

Hooks from user, project, local-project, managed, and plugin settings are
merged. Hooks can also be placed in skill or subagent frontmatter and remain
active while that component runs. Tool hooks configured for the main session
also run for subagent tool calls, whose input includes agent-identifying fields.

!!! note "Codex"
    Codex implements the same event/matcher/handler shape for its current hook
    system. Its `PreToolUse` can inspect and block Bash, `apply_patch`, MCP, and
    most local function tools, and accepts the same structured `deny` shape or
    exit code `2`. Hosted tools do not all pass through the local hook path.

## Skills and custom commands

Custom slash commands are now represented by skills. A skill is a directory
containing `SKILL.md`; it may also contain scripts, templates, and reference
files. Project skills live under `.claude/skills/`, personal skills under
`~/.claude/skills/`, and plugins can supply more.

```markdown
---
name: check-release
description: Check whether a release is ready and report missing items.
argument-hint: <version>
disable-model-invocation: true
---

Check release `$ARGUMENTS` against the repository's release files.
```

The frontmatter controls the name shown in the command menu, description,
argument hint, permitted tools, model, context mode, and whether Claude may
invoke the skill automatically. `$ARGUMENTS` expands to all arguments;
`$ARGUMENTS[0]` and `$0` address individual arguments. A leading `!` command in
skill content can run shell context gathering before the resulting prompt is
sent to Claude.

`/reload-skills` re-scans skill and command directories during a running
session. `/plugin` manages installed plugins, and `/reload-plugins` applies
plugin changes without restarting when possible.

The separate [Agent Skills](../agent-skills/index.md) chapter covers the file
format and sharing one skill directory between agent products.

## Subagents

Subagents have their own context window, system prompt, tool selection, and
optional model. Definitions are Markdown files in `.claude/agents/` for a
project or `~/.claude/agents/` for a user:

```markdown
---
name: dependency-tracer
description: Trace dependency edges and report the files involved.
tools: Read, Grep, Glob
model: sonnet
---

Trace the requested dependency. Return file paths and the call chain.
```

Claude selects subagents from their descriptions or they can be named in a
prompt. `/subtask` explicitly starts a side task whose result returns to the
conversation. `/tasks` shows background work, and `/list-agents` lists agents
and sessions available for cross-session messaging when that feature is
enabled. In current releases `/agents` points to conversational or file-based
management of definitions rather than opening the older editor interface.

An `isolation: worktree` field gives a subagent a Git worktree. Hooks may be
declared in the agent frontmatter and apply only while it runs. The
[subagent reference](https://code.claude.com/docs/en/sub-agents) lists the
complete frontmatter schema and built-in agent types.

!!! note "Codex"
    Codex subagent definitions and orchestration controls use Codex
    configuration rather than `.claude/agents/`. Both products isolate model
    context; filesystem isolation is a separate worktree choice.

## MCP servers

`claude mcp` manages Model Context Protocol servers from the shell:

```sh
claude mcp list
claude mcp get github
claude mcp add --transport stdio local-tools -- ./bin/local-mcp
claude mcp add --transport http docs https://example.com/mcp
claude mcp remove docs
```

Inside a session, `/mcp` shows connection and authentication state.
`/mcp reconnect <server>`, `/mcp enable <server>`, and `/mcp disable <server>`
change it without opening the menu.

Servers can be configured at local, project, or user scope. Their tools appear
to Claude with names of the form `mcp__<server>__<tool>`, which is also the name
used by permission rules and hook matchers. MCP servers may additionally expose
resources and prompts; server prompts appear as commands in the form
`/mcp__server__prompt`.

!!! note "Codex"
    Codex also consumes MCP servers. Its server definitions are stored in Codex
    configuration, but the model-facing distinction between MCP tools,
    resources, and prompts is the same protocol distinction.

## A compact command index

This is the subset of commands that control the agent itself rather than
launching a particular bundled workflow:

| Area | Commands |
| --- | --- |
| Session | `/clear`, `/resume`, `/rename`, `/branch`, `/fork`, `/rewind`, `/background`, `/exit` |
| Long-running work | `/goal`, `/plan`, `/tasks`, `/subtask`, `/batch` |
| Context | `/context`, `/compact`, `/autocompact`, `/btw` |
| Model and usage | `/model`, `/effort`, `/fast`, `/status`, `/usage` |
| Files and Git | `/add-dir`, `/cd`, `/diff`, `/copy`, `/export` |
| Configuration | `/config`, `/memory`, `/permissions`, `/sandbox`, `/hooks` |
| Extensions | `/reload-skills`, `/plugin`, `/reload-plugins`, `/mcp` |
| Diagnosis | `/doctor`, `/debug`, `/feedback`, `/help` |

Commands implemented as bundled skills or workflows—including `/code-review`,
`/security-review`, `/deep-research`, `/run`, and `/verify`—use the same command
menu but are prompts and orchestration packages rather than primitive session
controls. The command reference marks which entries are built-ins, skills, or
dynamic workflows.

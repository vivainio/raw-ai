---
icon: lucide/notebook-pen
---

# Agent Memory

`CLAUDE.md` is memory you write. Auto memory is memory the agent writes —
notes it takes about you, the project, and its own past mistakes, saved
between sessions and read back into context on its own, without you asking.
The [previous chapter](../agent-dialogue/index.md) described the system
prompt as folding in "user-supplied instructions" alongside the harness's
own text. Auto memory is what happens when some of those instructions are
authored by the agent itself, based on what it observed you say and do in
an earlier session.

## Where it lives

Same directory scheme as the session logs from the last chapter: one folder
per project, keyed by the working directory path with slashes turned into
dashes.

```
~/.claude/projects/<project-slug>/memory/
```

For this book's own repo, that's `-home-v-r-raw-ai`, and right now it holds
exactly two files:

```
$ ls ~/.claude/projects/-home-v-r-raw-ai/memory/
MEMORY.md
feedback_push_aggressively.md
```

One index file, plus one file per memory. That split matters: the index is
small enough to sit in context on every single turn, while the individual
memory files are only pulled in when something in the conversation looks
relevant to them.

## The index

`MEMORY.md` is not a memory itself — it's a table of contents, one line per
memory, kept short on purpose:

```markdown
- [Push aggressively](feedback_push_aggressively.md) — commit+push each raw-ai change without asking first
```

That single line is what's always loaded. It's cheap enough to carry in
every request the way the system prompt is, and it's enough for the agent
to recognize "this situation matches something I already know" and go read
the real file.

## A memory file

Here's `feedback_push_aggressively.md` in full — the actual file behind
that index entry, and, not coincidentally, the reason every chapter of this
book has been getting committed and pushed without a confirmation prompt:

```markdown
---
name: feedback-push-aggressively
description: "In the raw-ai repo, commit and push each change immediately without asking for confirmation each time"
metadata:
  node_type: memory
  type: feedback
  originSessionId: 4a68cacc-858b-4a4d-a02d-c0e45eb12903
---

Push aggressively in the raw-ai repo — commit and push after each
edit/chapter without pausing to ask for confirmation first.

**Why:** User explicitly said "aggressively push in this repo" after I'd
been asking before each push. This is a personal public docs/book repo
(vivainio/raw-ai, a Zensical-based web book), low stakes to push directly.

**How to apply:** For raw-ai specifically, treat "commit and push" as
pre-authorized for routine content/chapter changes (new pages, edits, nav
updates). Still use judgment for anything unusual (force-push, history
rewrite, deleting content) — this covers normal forward commits only.
Don't generalize this to other repos without similar explicit instruction.
```

The shape is deliberate. `description` in the frontmatter does the same job
a skill's `description` does — it's the field the agent matches against the
current situation to decide whether this file is worth pulling into
context at all. `metadata.type` sorts it into one of a small set of
categories. And the body isn't just the rule; it's the rule plus **why**
the user gave it, plus **how** to apply it going forward — so a future
session can extrapolate to a case that wasn't explicitly spelled out,
instead of pattern-matching the wording too literally.

## Four kinds of memory

- **feedback** — corrections and confirmations about *how* to work.
  "Don't do X", but also "yes, that unusual approach was right" — both are
  worth keeping, since only recording corrections quietly drifts the agent
  away from approaches you already validated. This chapter's example is a
  feedback memory.
- **project** — facts about the work itself that aren't derivable from the
  code: a deadline, a migration's real motivation, a freeze window.
- **user** — who you are, your role, what you already know — enough to
  calibrate how deep an explanation should go.
- **reference** — pointers into systems outside the repo: "bugs are tracked
  in Linear project X," "oncall watches this Grafana board."

What's deliberately excluded is anything the repo already answers on its
own: code conventions, architecture, file layout, git history, debugging
war stories. Those are supposed to be re-derived by reading the code or
running `git log`, not fossilized in a memory file that can drift out of
sync with it.

## Linking memories together

Memory files can reference each other with a `[[name]]` wiki-link, matching
the `name` in another file's frontmatter — a `feedback` memory pointing at
the `project` memory that explains *why* the rule exists, say. The link is
allowed to point at a memory that doesn't exist yet; that's not a broken
link, it's a note that the topic is worth writing up properly later.

## Writing is two steps, reading isn't blind trust

Saving a memory means writing the file *and* adding its one-line pointer to
`MEMORY.md` — skip the index update and the memory exists on disk but
never gets found. Reading one back isn't treated as ground truth either: a
memory that names a specific file, function, or flag is a claim about what
existed when it was written, not a guarantee about now. Before acting on
one, the expected move is to check the thing it names still exists — grep
for the function, confirm the file's still there — the same way you'd
distrust a comment that's older than the code around it.

That asymmetry is the whole point of the design: cheap to keep around
(one index line), self-describing enough to know when it applies (the
`description` field), and never fully trusted without a quick check against
whatever's actually true right now.

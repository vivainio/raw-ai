---
icon: lucide/wrench
---

# Agent Skills

A skill is just a folder with a `SKILL.md` in it — a name, a description, and
instructions the agent reads when the description matches what you're asking
for. The folder can also hold scripts, templates, or reference docs the skill
needs.

```
my-skill/
├── SKILL.md
├── scripts/
│   └── helper.sh
└── reference.md
```

`SKILL.md` starts with frontmatter, then plain instructions:

````markdown
---
name: my-skill
description: One line describing what this does and when to use it.
---

# My Skill

Run the bundled script instead of reimplementing this logic inline:

```sh
bash scripts/helper.sh <arg>
```
````

The `description` is the only part the agent sees before deciding to load the
skill, so make it specific: what the skill does *and* the triggers that should
cause it to fire. Everything else in `SKILL.md` — including the path to
`scripts/helper.sh` — is instructions the agent follows once it's decided to
use the skill, the same way it'd follow any other command you gave it.

## One folder, not one per project

Skills don't have to live inside the project they're used from. Claude Code
looks for them in `~/.claude/skills/` — a single, home-directory-wide folder —
in addition to any project-local `.claude/skills/`. Put a skill there once and
every project on the machine can use it.

```sh
mkdir -p ~/.claude/skills/my-skill
```

Nothing stops you from keeping the actual files somewhere else — a dotfiles
repo, a shared skills repo checked out separately — and symlinking them in:

```sh
ln -s ~/code/my-skills-repo/my-skill ~/.claude/skills/my-skill
```

That's how a lot of real skill directories look in practice: not real
directories at all, just links pointing at wherever the skill's source of
truth actually lives.

```sh
$ ls -la ~/.claude/skills/
aws-cli -> /home/v/ai/skills/aws-cli
wrangler -> /home/v/.cache/skillset/repos/some-org/skills/wrangler
```

## Sharing one skill set across agent tools

Agent Skills is an open spec, but each tool still picks its own home-directory
folder for personal, cross-project skills: `~/.claude/skills/` for Claude Code,
`~/.codex/skills/` for Codex CLI, `~/.copilot/skills/` for Copilot,
`~/.gemini/skills/` for Gemini CLI. Rather than keep separate copies in sync,
pick one as the real folder and symlink the rest to it:

```sh
ln -s ~/.claude/skills ~/.codex/skills
ln -s ~/.claude/skills ~/.copilot/skills
```

Every tool now resolves to the same files on disk. Edit a `SKILL.md` once and
every tool sees the change immediately.

!!! note "Project-only skills"
    If a skill is genuinely specific to one project — it references that
    project's internal tooling, say — keep it in a real (non-symlinked)
    `.claude/skills/` inside the repo instead. Reserve the home-directory
    store for skills you'd want anywhere.

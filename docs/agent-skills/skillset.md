# Managing skills with skillset

Keep skills in topic repos, not loose in `~/.claude/skills/` — a `jira-skills`
repo, a `cloudflare-skills` repo, your own personal grab-bag. `~/.claude/skills/`
is then just symlinks into those repos, never edited directly. This is the
starting point, not something to graduate into later.

Managing those symlinks by hand — cloning, finding the right subfolder,
linking, remembering to pull updates — is what
[skillset](https://github.com/vivainio/skillset) is for. It clones/caches skill
repos, links the skills you want, and searches across everything cached
without you tracking which repo a skill came from.

## Installing

```sh
skillset add vivainio/agent-skills          # clone, pick skills interactively
skillset add vivainio/agent-skills -s zaira # install just one
```

Installs to the project scope if a `skillset.yaml` exists in the current
directory or a parent; otherwise falls back to `~/.claude/skills/`. `-g` forces
global.

## Searching

```sh
skillset search jenkins
skillset search doc-%     # glob
```

Searches everything cached locally, offline — which means you can stock up on
repos before you know which skill you want:

```sh
skillset add vivainio/agent-skills --fetch   # clone/cache only, link nothing
skillset add some-org/other-skills --fetch
```

Then later, once you remember you need something:

```sh
skillset search invoice
```

turns up matches across every repo you've fetched, without you having to
recall which one it was in.

## Removing

```sh
skillset remove zaira                    # unlink one skill
skillset remove vivainio/agent-skills    # drop the whole repo
```

## Declaring the set in skillset.yaml

For a reproducible setup across machines or a team, declare it instead of
installing ad hoc:

```yaml
skills:
  vivainio/agent-skills:
    enabled: ["*"]

  vivainio/jira-skills:
    enabled: [bu-jira-process]
    disabled: [experimental-jira-thing]
```

```sh
skillset update    # pulls each repo, links enabled, unlinks disabled
```

`skillset init` creates this file (`-g` for the global one). Check it into the
repo and a fresh clone gets the same skills back with one `skillset update`.

!!! note "Symlinks, not copies"
    Skills come from skillset's repo cache (`~/.cache/skillset/repos/`), not
    copies. Push an update to the source repo and it's live everywhere it's
    linked.

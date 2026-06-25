# `rasa.module.releases` — content

This is what the module ships. It is the portable release-management
capability distilled from `rasa.domain.code`, with the two things that
vary per project — *what makes a release shippable* and *how it actually
deploys* — pulled out into a project-owned `release-gate.md`.

## What installs where

| Source | Installs to | Policy | Owner |
|---|---|---|---|
| `content/release-rules.md` | `.claude/release-rules.md` | file-replace | Element (refreshed on upgrade) |
| `content/skills/{release,release-add,releases}/` | `.claude/skills/` | directory-mirror | Element |
| `seed/release-gate.md.template` | `.claude/release-gate.md` | skip-if-exists | **Project** (you fill it) |
| `seed/releases/RELEASES.md.template` | `releases/RELEASES.md` | skip-if-exists | Project |
| `seed/rasa.lock.json.template` | `.claude/rasa.lock.json` | init-only-with-sha | Project |

## The split that makes it portable

- **Element-owned (`content/`)** — the release lifecycle, the tracker
  format, the version heuristic, the hotfix path, the dependency-audit
  cadence, and the three skills. Identical for every engineering project.
  Upgrades flow in.
- **Project-owned (`seed/`)** — the `release-gate.md` (the readiness
  checks + the deploy command) and the `releases/RELEASES.md` timeline.
  The project fills the gate; the tracker accumulates its own history.
  Never overwritten on upgrade.

The single varying concern — *what "shippable" means and how this project
deploys* — lives in `.claude/release-gate.md`, the release analogue of
the task module's done-gate.

## Soft cross-references (siblings, not dependencies)

Every cross-module reference is a convention that degrades gracefully:

- **`rasa.module.tasks`** — release entries list `TASK-NNN` ids; without
  it, freeform descriptions work fine.
- **`rasa.module.pipelines`** — the deploy command can point at the
  pipeline's `50-deploy` stage; without it, name a raw command in the
  gate.
- **`rasa.module.tests`** — the "tests green" check can run a named
  suite; without it, a raw test command.

## Mounting into a parent

`rasa.module.releases` is a `module` (canon Spec §6): a focused
capability mountable into a parent `domain` or `orchestrator`. A parent
opts in via its own `rasa.json`:

```json
"requires": {
  "elements": [
    { "name": "rasa.module.releases", "version": ">=0.1.0" }
  ]
}
```

The kernel resolver pulls it in dependency order; `bin/init` installs its
`content/` + `seed/` per the table above.

## Skills

- **`/release`** — cut a production release end-to-end (preflight → merge
  → tag → deploy → push → record). Invocation is consent; hard-stops only
  on real blockers.
- **`/release-add`** — append a task/item to the top "🚧 Next" entry of
  the tracker. Idempotent; `--since-last-tag` bulk mode.
- **`/releases`** — read + render the release timeline. Read-only.

## See also

- `content/release-rules.md` — the release spine (the contract).
- `seed/release-gate.md.template` — the per-project readiness + deploy
  adapter.
- `~/rAI/rasa-os/elements/module-tasks/` — the first module; same shape,
  task lifecycle.
- Canon Spec §6 — the `module` kind definition.

# CLAUDE.md — `rasa.module.releases`

> **Who you are (SA-025).** `rasa.module.releases` — the RasaOS module for releases. Substrate: **RasaOS**; role: **module**. On install `bin/init` renders this into `.claude/rasa-identity.md`; `/whoami` composes the full identity with the project's deployment layer.


Per-repo working contract for Claude sessions opened inside this folder.
Extends `~/.claude/CLAUDE.md` and the workspace `~/rAI/rasa-os/CLAUDE.md`
(the `rasa.tenant.rasaos` tenant's contract); does not override them.

## What you are when you're in this folder

You are working on **`rasa.module.releases`** — an engineering-scoped
`module`-kind Element: the release-management lifecycle distilled out of
`rasa.domain.code` and reshaped so it mounts into any parent domain or
orchestrator. It is one of four engineering modules distilled together
(releases, pipelines, tests, jobs), siblings to the first module,
`rasa.module.tasks`.

A `module` (canon Spec §6) is "a focused capability that extends a domain
or orchestrator, mountable into one or more parents." Parents pull it in
via their own `requires.elements[]`.

## The load-bearing idea: the release-gate

This module is deliberately **engineering-shaped** — git, tags, semver,
CI, dependency audits are all fair game (release management *is* that).
The discipline is different from `module.tasks`: there, the goal was
domain-neutrality. Here, the goal is to not hardcode the slice that
varies **between engineering projects**:

1. **What makes a release shippable** — the test / build / audit checks.
2. **How this project deploys** — the deploy command + tag-env model.

Both live in the project-owned `.claude/release-gate.md` (seeded
`skip-if-exists`). `content/` references the gate; it hardcodes neither.

If you find yourself about to bake `npm run deploy` / `fastlane` /
`firebase` / a specific test command / a specific env name into
`content/`, **stop** — that belongs in the release-gate or, when the
pipelines module is mounted, in its deploy stage.

## Soft cross-references — keep them soft

This module names its siblings (`module.tasks` for `TASK-NNN` ids,
`module.pipelines` for the deploy stage, `module.tests` for suites). Every
such reference is a **convention that degrades gracefully** — present-if-
mounted, freeform-if-absent. Do **not** turn a soft reference into a hard
`requires.elements[]` dependency: a release module that can't track a
release without the tasks module mounted has failed its one job.

## Source of truth

- **`~/rAI/rasa-os/canon/`** — authoritative. Spec §6 defines the
  `module` kind; ELEMENT_CONTRACT.md §7 defines the install policies; the
  schema enforces that only `module` may declare `requires.parent_kind`.
- **`content/release-rules.md`** — the release spine. The contract for
  every release in any consuming project.
- **`content/README.md`** — what installs where, Element- vs project-
  owned.
- **`rasa.json`** — the formal declaration + install manifest.

## Don'ts

- **Don't re-couple the spine to one project's deploy.** No
  `npm`/`fastlane`/`firebase`/`kubectl`/specific-test-command assumptions
  in `content/`. They go in the release-gate.
- **Don't harden a soft reference into a hard dependency.** Siblings are
  conveniences, not requirements.
- **Don't claim paths a parent owns.** A module mounts under a parent that
  already has its own `CLAUDE.md`, output styles, etc. This Element seeds
  only what is release-specific (`release-gate.md`, the
  `releases/RELEASES.md` tracker).
- **Don't `bin/init` this Element into itself.** `content/` is the
  source; copying it into this repo's `.claude/` would duplicate it.
- **Don't absorb general git plumbing.** Branch/merge discipline lives in
  the project's `git-flow-rules.md` (or a future git module); this module
  is the release lifecycle, not the whole VCS contract. `/push`-style
  general pushing stays out of scope.
- **Don't push from the Cowork sandbox.** Local commit + tag only; the
  user pushes from their machine (workspace rule).

## How a version bump works

- **Patch (0.1.0 → 0.1.1)** — wording fix, template clarification,
  `bin/*` bug fix. No structural change.
- **Minor (0.1.x → 0.2.0)** — new skill, new seed file, a new capability.
  Parents may adopt; not breaking.
- **Major (0.x.x → 1.0.0)** — first stable lock-down, or a breaking change
  to the lifecycle / tracker format / install shape after 1.0. Parents
  REQUIRED to migrate.

Each bump: edit `VERSION` + `rasa.json#version`, write a CHANGELOG entry,
run `bin/check-manifest`, commit + tag. Add a row to
`~/rAI/rasa-os/elements/CHANGELOG.md` (track #2) and update
`~/rAI/rasa-os/elements/REGISTRY.md`.

## What success looks like

- An engineering project can mount this module, fill one
  `release-gate.md`, and run the full release lifecycle — cut, track,
  view — with no edits to `content/`.
- `rasa.domain.code` could, in time, consume this module + the pipelines/
  tests modules instead of carrying the whole release/CI system inline —
  the distillation proving itself in reverse.

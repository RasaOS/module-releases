# RasaOS Module · Releases

**Canonical name:** `rasa.module.releases`
**Repo / folder:** `module-releases`
**Kind:** `module` (canon Spec §6 — *a focused capability that extends a domain or orchestrator, mountable into one or more parents*)
**Contract:** Element Contract v1.3.0
**Version:** 0.1.0
**Status:** Live. Engineering-scoped release module; companion to `rasa.module.tasks`.

## What this is

A portable release-management lifecycle that mounts into **any** domain
or orchestrator. It is the release system distilled out of
`rasa.domain.code` — the version-bump heuristic, the `RELEASES.md`
timeline tracker, annotated-tag discipline, the hotfix path, and
dependency-audit cadence — with the two genuinely per-project concerns
pulled out into a project-defined **release-gate**.

```
releases/RELEASES.md  🚧 Next (accumulating)  →  /release  →  ✅ Shipped (tagged)
        ↑ /release-add appends as work merges to main          ↓ next 🚧 opens
```

## Engineering-scoped, with seams

Unlike `rasa.module.tasks` (which is domain-neutral so legal/health/
writing domains can mount it), this module is **frankly engineering-
shaped** — it speaks git, tags, semver, CI, and dependency audits,
because release management *is* that. What it refuses to hardcode is the
slice that genuinely varies between engineering projects:

- **What makes a release shippable** — which test / build / audit
  commands must be green.
- **How this project deploys** — the actual deploy command + tag-env
  model.

Both live in the project-owned **`.claude/release-gate.md`** (seeded
`skip-if-exists`), the release analogue of the task module's done-gate.

## The release-gate — how it adapts per project

`/release` reads `.claude/release-gate.md` at pre-flight. It declares:

- **Web** → `npm test` + `npm run build` green; `npm run deploy:prod`.
- **iOS** → `xcodebuild test` green; `fastlane release`; appstore tag.
- **Container** → `make test` + image greenlight; `50-deploy.sh --env prod`.

`release-rules.md` references the gate; the lifecycle, tracker, and
version discipline stay identical everywhere.

## Plays well with its siblings (soft references)

- **`rasa.module.tasks`** — release entries list `TASK-NNN` ids; freeform
  works without it.
- **`rasa.module.pipelines`** — the deploy command points at the
  pipeline's `50-deploy` stage when mounted.
- **`rasa.module.tests`** — the "tests green" check can run a named suite.

Every cross-reference is a convention that degrades gracefully — no hard
dependency.

## Install / mount

A parent domain or orchestrator opts in via its own `rasa.json`:

```json
"requires": {
  "elements": [
    { "name": "rasa.module.releases", "version": ">=0.1.0" }
  ]
}
```

The kernel resolver pulls it in dependency order; the declarative install
applies `element.files[]` + `seed.files[]`. For local install testing,
`bin/init <target-dir>` copies the content per the manifest. See
[`content/README.md`](content/README.md) for the full file-by-file map.

## Layout

- `content/release-rules.md` — the release spine (Element-owned).
- `content/skills/` — `/release`, `/release-add`, `/releases`.
- `seed/release-gate.md.template` — the per-project readiness + deploy adapter.
- `seed/releases/RELEASES.md.template` — the live timeline (project-owned).
- `bin/init`, `bin/check-manifest` — the canonical installer + manifest checker.

## See also

- `~/rAI/rasa-os/elements/domain-code/` — the engineering domain this was distilled from (keeps its own inline release content).
- `~/rAI/rasa-os/elements/module-tasks/` — the first module; the shape this follows.
- Canon Spec §6 — the `module` kind.
- `~/rAI/rasa-os/elements/REGISTRY.md` — live Element registry.

# CHANGELOG — `rasa.module.releases`

Reverse-chronological. Each entry is a version bump.

---

## 0.1.0 — 2026-06-24 — INITIAL

**Engineering-scoped release module — one of four (releases, pipelines,
tests, jobs) distilled together from `rasa.domain.code`.** A portable
release-management lifecycle reshaped to mount into any parent domain or
orchestrator.

### What it is

- **Kind:** `module` (canon Spec §6) — opt-in, mountable into a parent
  `domain` or `orchestrator` via the parent's `requires.elements[]`.
  `requires.parent_kind: [domain, orchestrator]`.
- **Contract:** Element Contract v1.3.0. `bin/check-manifest` GREEN;
  conforms to `RasaOS/schema` v0.1.0.

### Distilled from `rasa.domain.code`

The portable release core that travels:

- **Release tracker** — `releases/RELEASES.md`: one "🚧 Next" accumulator
  at the top, "✅ Shipped" entries below stamped with date / tag / sha.
  (Path moved from domain-code's `tasks/RELEASES.md` to a releases-owned
  `releases/` dir, so the module stands alone without the tasks module.)
- **Version-bump heuristic** — patch / minor (default) / major semver.
- **Production tag discipline** — annotated tags only, `vX.Y.Z` with an
  optional `-<sha>-<env>` stamp, lowercase `v`, bootstrap rule, rollback
  semantics.
- **Hotfix path** — `hotfix/HOTFIX-NNN-slug`, single-concern, patch-bump,
  🔥 audit entry, postmortem within 48h.
- **Dependency hygiene** — gated manifest edits, on-release audit cadence,
  quarterly sweep.
- **Skills** — `/release` (cut end-to-end), `/release-add` (append,
  idempotent, `--since-last-tag`), `/releases` (read + render; net-new
  viewer paralleling the tasks module's `/backlog`).

### The release-gate (the load-bearing adapter)

Per the user's "engineering-scoped + seams" decision, the module keeps
engineering vocabulary but delegates the two genuinely per-project
concerns to a project-owned adapter:

- `.claude/release-gate.md` (seeded `skip-if-exists`) declares (1) the
  readiness checks — which test / build / audit commands must be green —
  and (2) the deploy command + tag-env model. `/release` reads it at
  pre-flight and hard-stops if a check fails or the deploy command is
  missing. Ships with a minimum-honest default + commented web / iOS /
  container examples. The release analogue of `module.tasks`' done-gate.

### Soft cross-references (siblings, not dependencies)

- `module.tasks` → `TASK-NNN` ids in the tracker (freeform without it).
- `module.pipelines` → the deploy command points at `50-deploy.sh`.
- `module.tests` → the "tests green" check runs a named suite.

All present-if-mounted, graceful-if-absent. No hard `requires.elements[]`.

### Install shape

- **Element-owned (`element.files[]`, refreshed on upgrade):**
  `release-rules.md`, `skills/{release,release-add,releases}/`.
- **Project-owned (`seed.files[]`, `skip-if-exists`):** `release-gate.md`,
  `releases/RELEASES.md`, and the stamped `rasa.lock.json`.

### Provenance / decisions

- Source: `rasa.domain.code` v0.42.0 `content/release-rules.md` and the
  `release` / `release-add` skills. `domain-code` keeps its
  engineering-specific release content inline; it may later consume this
  module.
- **Boundary decision:** general git plumbing (`/push`, branch/merge
  discipline) stays out — this module is the release lifecycle, not the
  whole VCS contract. The deploy *mechanics* belong to `module.pipelines`;
  this module orchestrates the release *around* the deploy.

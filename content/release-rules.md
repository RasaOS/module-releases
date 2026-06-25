# Release Rules

The portable release-management spine that `rasa.module.releases` installs
at `.claude/release-rules.md`. It covers release tracking, version-bump
heuristics, production tag format, the hotfix path, and dependency
hygiene. **Read this file when shipping a release or auditing
dependencies.**

This file is **Element-owned** — it refreshes on upgrade. It deliberately
does **not** decide the two things that vary per project:

1. **What makes a release shippable** (which tests, which build, which
   audit must be green), and
2. **How this project actually deploys** (the deploy command, the
   environment + tag-stamp model).

Both live in the project-owned **`.claude/release-gate.md`** (the release
analogue of the task module's done-gate). This file references the gate;
it hardcodes neither. Where a `.claude/git-flow-rules.md` exists, its
branch/merge/deploy-authorization rules also apply to every release.

## Soft cross-references (degrade gracefully)

This module stands alone, but plays well with its siblings when they are
mounted. Every cross-reference below is **soft** — a convention, not a
hard dependency:

- **`rasa.module.tasks`** — release entries list `TASK-NNN` / `HOTFIX-NNN`
  ids. If the tasks module isn't mounted, use freeform one-line
  descriptions instead; nothing breaks.
- **`rasa.module.pipelines`** — when present, the deploy command in your
  release-gate points at the pipeline's deploy stage
  (`build/stages/50-deploy.sh`). When absent, the gate names a raw deploy
  command directly.
- **`rasa.module.tests`** — the "tests green" readiness check in your
  release-gate runs the project's test command (a test suite from the
  tests module, or a raw command).
- **Audit ledger** — release events record to the project's audit ledger
  if it has one (`tasks/AUDIT.md` from the tasks module, or the project's
  own `AUDIT.md`); otherwise `releases/RELEASES.md` is the sole record.

## Release tracking — `releases/RELEASES.md`

`releases/RELEASES.md` is the **live release tracker** (project-owned,
seeded once): a single timeline of releases, with the current "next"
entry at the top accumulating work as it merges to `main`, and shipped
entries below stamped with date + tag.

Items **auto-append** to the top "🚧 Next" entry when they land on
`main` (via a review-merge skill, via `/release`'s integration merge, or
by invoking `/release-add` after a manual merge). The user does not
maintain this file by hand.

### Format

```markdown
# Releases

🚧 v0.38.0  ◆  next release — accumulating since v0.37.0

- TASK-042 — fix login bug on iOS
- TASK-043 — add filter UI to the inbox
- HOTFIX-007 — patch RBAC bypass on /admin

---

✅ v0.37.0  ◆  /sync-all — autonomous variant of /sync
              shipped 2026-05-20 · tag v0.37.0 · sha f77a843

- TASK-040 — /sync-all skill
- TASK-041 — /sync When-NOT cross-reference

---

✅ v0.36.0  ◆  status overhaul — `blocked` added (BREAKING)
              shipped 2026-05-20 · tag v0.36.0 · sha 8274f0f

- TASK-038 — done → completed rename
- TASK-039 — blocked state + Blocker discipline
```

Format conventions, line by line:

- **Status glyph + version + diamond + summary line.** `🚧` means "in
  progress" (accumulating); `✅` means "shipped." The diamond `◆`
  separates the version from the one-line summary. The summary is the
  *release theme* — what makes this version coherent — not a task list.
- **Detail line** (shipped only). `shipped <YYYY-MM-DD> · tag <tag> · sha
  <short-sha>`. Indented two spaces under the version line.
- **Item list.** One bullet per unit of work that landed. `TASK-NNN —
  <title>`, `HOTFIX-NNN — <title>`, or a freeform `<title>` when the
  tasks module isn't in play. Order is merge order (oldest first).
- **`---` separator** between entries.

### The lifecycle of an entry

Three states, but only two appear in the file at any given time: the
**one** "🚧 Next" entry at the top, and the **N** "✅ Shipped" entries
below.

1. **🚧 Next** — the active accumulator. Always exactly one, at the top.
   The version number is the *next expected* version (last shipped +
   default bump). Items append as they merge.
2. **✅ Shipped** — terminal. `/release` flips the "🚧 Next" entry to
   "✅ Shipped" at ship time, fills in date / tag / sha, then creates a
   **new** "🚧 Next" entry above it for the following release.

### How items land in the "🚧 Next" entry

Auto-append happens on **merge to `main`** — the moment work becomes part
of "what will ship next." Three paths land here:

1. A **review-merge** skill (e.g. `/peer-review`) → calls `/release-add`
   with the merged `TASK-NNN` / `HOTFIX-NNN`.
2. **`/release` merges an integration branch** → calls `/release-add` for
   every id in the integration's commits.
3. **Manual `git merge` / `gh pr merge`** (no skill involved) → the user
   runs `/release-add` after the fact, or `/release-add --since-last-tag`
   to bulk-catch up.

`/release-add` is **idempotent** — re-running it for the same id is a
no-op.

### How `/release` uses the tracker

`/release` reads the "🚧 Next" entry as the **release manifest**. At ship
time it: (1) cross-checks the entry against what actually merged (commit
log since last tag), flagging and fixing drift; (2) stamps the entry
`✅` with date / tag / sha and the release theme; (3) creates the next
"🚧 Next" entry above; (4) commits the change as part of the release.

`releases/RELEASES.md` is the *release-shaped* record; the project's
audit ledger is the *chronological* record. Complementary, not redundant.

## Production deploy tagging (mandatory)

Every successful production deploy from `main` is tagged. Tags are the
version-controlled record of what shipped to users and when — `git log
--tags` becomes the deploy history.

### Format

Release tags carry the full build stamp: `v<MAJOR>.<MINOR>.<PATCH>` with
an optional `-<shortsha>-<env>` suffix — e.g. `v1.2.0` or
`v1.2.0-9f3a1c7-prod`. The semver is what the release closer proposes and
the reviewer confirms; the `-<sha>-<env>` suffix, when used, records
exactly which commit shipped and to which environment. How the suffix is
produced is project-specific — your `release-gate.md` (or, if mounted,
the pipelines module's `environment` helper) defines it. Always lowercase
`v`. **Annotated** tags only — never lightweight — so the message can
carry release notes.

### Bootstrap

The first tagged release is **`v1.0.0`** (or `v0.1.0` for pre-1.0
projects — match the project's existing convention). History before this
rule existed is "pre-versioning" and is not retroactively tagged.

### Version-bump heuristic

The closer proposes a version with reasoning; the reviewer confirms or
overrides. Do not ship without an agreed version.

- **Patch** (`vX.Y.Z+1`) — bug fixes, copy / styling tweaks, no new
  user-visible features.
- **Minor** (`vX.Y+1.0`) — new user-visible features, additive changes.
  The default for most batches.
- **Major** (`vX+1.0.0`) — breaking changes (removed features, schema
  migrations affecting existing users, route changes). Rare. Always
  paired with a user-facing note.

### Deploy + tag flow (after "yes, ship")

The deploy command is project-specific — it lives in
`.claude/release-gate.md` (or delegates to the pipelines module's deploy
stage). `/release` orchestrates the full sequence. The shape:

```sh
<deploy command per .claude/release-gate.md>   # e.g. the pipelines 50-deploy stage,
                                               #      npm run deploy, fastlane release, …
# After deploy succeeds — build the tag, then tag and push:
git tag -a "vX.Y.Z" -m "<release notes>"       # on main HEAD (annotated)
git push origin "vX.Y.Z"
```

Release-note message format (annotated tag body, multi-line):

```
vX.Y.Z — <one-line summary>

Shipped:
- TASK-NNN — <name>
- TASK-NNN — <name>

Deployed: <YYYY-MM-DD HH:MM UTC>
Integration: <branch or PR>
```

### Closing report after deploy

Include in the deploy completion report: the tag with a clickable link to
its release page (`https://github.com/<owner>/<repo>/releases/tag/<tag>`),
confirmation the tag was pushed to origin, and the merge-commit SHA the
tag points to.

### Rollback semantics

The project's rollback command (per `release-gate.md`) reverts the live
build but does **not** move git tags. If a tagged release is rolled back,
the tag stays as a historical record of what was deployed, and a new tag
(a patch bump) marks the restored version. Document the rollback in the
new tag's message.

## Hotfix path (emergencies)

The normal flow assumes the happy path: a batch of work stabilized,
integration-tested, then shipped. When production is broken, that flow is
too slow. The hotfix path exists for "fix is needed *now*, the queue can
wait."

### When to invoke

- Production is bricked or in a clearly-bad state.
- A deployed bug is causing data loss, security exposure, or
  user-blocking errors.
- A regression caught in stage that would block the next deploy.

If the bug is annoying-but-not-urgent, **don't hotfix.** File a normal
task and ship in the next batch. Hotfix is a privilege that costs queue
discipline; spend it carefully.

### Flow

1. **Branch from `main`.** `hotfix/HOTFIX-NNN-slug` (HOTFIX numbering is
   independent of TASK numbering — restart at 001 per project, increment
   per incident).
2. **Single concern.** A hotfix branch fixes exactly one thing. No "while
   I'm here" changes — bundling defeats the speed argument.
3. **The readiness gate still applies.** Whatever `.claude/release-gate.md`
   requires (tests green, build clean) is still required. Hotfix does not
   mean "skip the gate." If the gate is red *because of the bug*, surface
   it — repair it as part of the hotfix, don't bypass.
4. **PR opens with title `HOTFIX-NNN: <summary>`** and a body explaining
   symptom, root cause, fix, and verification.
5. **Deploy is patch-bump only.** `vX.Y.Z` → `vX.Y.Z+1`. Major/minor
   bumps imply scope; hotfixes are scope-disciplined patches.
6. **Mark the audit entry with 🔥** so future readers can find the
   incident chain at a glance:
   ```
   - 🔥 **Hotfix HOTFIX-NNN — <summary>.** Released vX.Y.Z+1.
     Postmortem at docs/postmortems/YYYY-MM-DD-….md.
   ```
7. **Postmortem follows.** Every hotfix pairs with a postmortem within 48
   hours. The point is the lesson, not the absolution.

### What hotfix does NOT change

- The readiness gate is still the contract.
- The deploy-tagging rule still applies (annotated tag, pushed, audit
  entry).
- **Invocation is consent.** Typing `/release` on a hotfix branch IS the
  deploy authorization — the route changes (skip the integration buffer,
  default to a patch bump) but not the authorization model. The skill
  still hard-stops on real blockers (failed gate, missing deploy command).

## Dependency hygiene

Dependencies drift; drift becomes vulnerabilities. This rule keeps the
project honest about its dependency surface without turning dependency
management into a daily chore.

### Adding / upgrading / removing dependencies

- Touching the project's manifest file(s) (`package.json`, `Cargo.toml`,
  `go.mod`, `Gemfile`, `pyproject.toml`, `Package.swift`, etc.) is a
  **gated** action — the user approves any add/upgrade/remove.
- The same change commits the lockfile update (`package-lock.json`,
  `Cargo.lock`, `go.sum`, etc.). Lockfile out of sync with manifest is a
  blocker.
- Major-version upgrades require explicit acknowledgment — they often
  ship breaking changes that need a release-notes scan.

### Audit cadence

- **On every release** (every `/release`): run the project's audit
  command (`npm audit`, `cargo audit`, `pip-audit`, etc. — named in
  `release-gate.md`). Surface any HIGH / CRITICAL findings in the closing
  report. Don't auto-block on advisories — the user decides ship-then-
  patch vs fix-first.
- **Quarterly sweep** — once per quarter, a full dependency audit +
  targeted upgrade pass, filed as a task so it's tracked.

### Linkage to the audit ledger

Significant dependency events get an audit entry: major-version upgrades
of a foundation dependency, security patches for HIGH/CRITICAL
advisories, swapping out a dependency. Routine patch-level upgrades ride
on the release entry — no separate audit entry needed.

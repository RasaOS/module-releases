---
name: release
description: Cut a production release end-to-end — preflight, merge integration into main, tag the release commit, deploy, push the tag, record. Reads the project's .claude/release-gate.md (and CLAUDE.md) to discover the actual deploy command and readiness checks. **Invocation is consent: this skill does not ask for confirmation at any soft gate.** It stops only at hard blockers (auth, failed gate, dirty tree, missing deploy command, etc.). Triggered when the user wants to ship — e.g. "/release", "/release patch", "/release v1.2.0", "ship it", "cut a release".
---

# /release — Cut a production release

Orchestrate the deploy sequence end-to-end. Discover the project's deploy
command and readiness checks from `.claude/release-gate.md` rather than
hardcoding, then run the canonical sequence: **preflight → merge
integration → tag → deploy → push tag → record**.

This skill executes the contract in `.claude/release-rules.md`; it does
not redefine it. If the two disagree, `release-rules.md` wins.

**Invocation is consent.** The user typed `/release` — that is the
release authorization. This skill does not ask "are you sure?", does not
ask "deploy now?", does not ask the user to confirm the version. It
**runs, or stops hard on a real blocker.** Where a
`.claude/git-flow-rules.md` exists, its Rule 2 carve-out (user-invoked
merge-bearing skills) and Rule 5 (deploys route through `/release`)
apply.

## The release-gate (the load-bearing adapter)

Everything that varies per project lives in `.claude/release-gate.md`:

- **Readiness checks** — what must be green to ship (the test command,
  the build command, the dependency audit). Each must be objectively
  checkable — a command that exits zero — never a feeling.
- **Deploy command** — how this project actually ships. If
  `rasa.module.pipelines` is mounted, this points at the pipeline's
  deploy stage (`build/stages/50-deploy.sh --env prod`). Otherwise it's a
  raw command (`npm run deploy`, `fastlane release`, …).
- **Tag-env model** — how the optional `-<sha>-<env>` tag suffix is
  produced, if used.

Read it first. Never invent a deploy command or a test command — if the
gate doesn't name one, hard-stop and ask the user to fill the gate.

## Behavior contract

- **Invocation is the user confirmation.** No "Deploy now?", "Confirm
  version?", or "Proceed through warnings?" prompts. The job is to
  discover, verify, execute, or stop hard on a real blocker.
- **Hard blockers stop the run.** A hard blocker is a *real* failure —
  not "we'd like to double-check." The full list:
  - Auth failure (no credentials, expired token, `gh`/cloud CLI not
    logged in).
  - Pre-flight failure (dirty working tree, behind upstream, branch in a
    state the deploy can't run from).
  - Readiness-gate failure (a check named in `release-gate.md` exits
    non-zero — tests, build, or audit).
  - Missing deploy command (`release-gate.md` names none, and none is
    discoverable from `CLAUDE.md` / manifests).
  - Release-plan mismatch (the `releases/RELEASES.md` "🚧 Next" entry
    declares scope that doesn't match what merged — surface and stop).
  - Merge refused by branch protection.
  - Deploy command itself fails — stop, no retry, report exactly what
    failed.
- **Pick the version; do not ask.** With a version arg (`/release patch`
  / `minor` / `major` / `v1.2.3`), use it. Otherwise compute it from the
  heuristic in `release-rules.md`. Log the choice as a flagged assumption
  in the closing report.
- **Order: preflight → merge → tag → deploy → push tag.** The tag is
  created on the merge commit *before* deploy runs, so the commit's
  identity is locked. The tag is pushed *after* deploy succeeds, so a
  failed deploy doesn't publish a stale release tag.
- **Platform routing is automatic.** If a `<platform>-release` skill
  exists for the detected platform, route to it; the closing report names
  which skill ran. Do not ask.
- **Annotated tags only.** `git tag -a` with release notes. Lightweight
  tags don't carry the message.
- **Record on ship.** The project's audit ledger gets a 🚀 entry (if one
  exists); `releases/RELEASES.md` flips the version's entry to ✅ Shipped
  with the tag. These edits are committed to `main` as part of the
  release — never left dangling.
- **Honest reporting on failure.** Partial state is the worst state to
  leave undocumented. Any failure mid-flow is captured in the closing
  report with the exact step, command, and error. No retry without
  explicit user re-invocation.

## The flow

### Step 0 — Platform detection + auto-routing

Determine the platform from an explicit `## Platform` declaration in
`CLAUDE.md`, else infer from manifests (`*.xcodeproj`/`Package.swift` →
`ios`; `package.json` with react-native → `react-native`; `package.json`
→ `web`; `pyproject.toml`/`setup.py` → `python`; gradle+android →
`android`; otherwise `universal`). If a `<platform>-release` skill exists
in `.claude/skills/`, **route to it automatically** and state so in the
report. Otherwise continue below.

### Step 1 — Pre-flight check

Run in parallel:

- `git rev-parse --abbrev-ref HEAD` — current branch.
- `git status --porcelain` — must be empty (no dirty files).
- `git fetch origin` — capture remote state.
- `git log HEAD..origin/main --oneline` — if non-empty AND on main, the
  local is behind → hard-stop "behind upstream."
- `git describe --tags --abbrev=0` — the previous tag.
- Each **readiness check** named in `.claude/release-gate.md` (test
  command, build command). Must exit 0 (or, for the build, unless the
  gate marks warnings as fatal).
- **Deploy command** resolvable from `release-gate.md`. Must be present.
- `releases/RELEASES.md` lookup — read the top "🚧 Next" entry. Cross-
  check against commits since the last tag:
  - **Items in commits, missing from the entry** → silently add them (a
    skipped `/release-add`); not a stop condition.
  - **Items in the entry, missing from commits** → hard-stop with the
    diff. The entry claims something that didn't merge.

Render the pre-flight summary as a compact status box (● = passed, ◐ =
running, ✗ = failed). Any ✗ → the run **stops** with an error block
naming the failed check, the exact reason, and "hard-stop per /release
contract."

### Step 2 — Pick version

With a version arg, use it (`patch`/`minor`/`major` bump from the
previous tag, or an exact `vX.Y.Z`). Otherwise compute the next version
from the `release-rules.md` heuristic (patch = fixes; minor = additive,
the default; major = breaking). State the choice as an informational
blockquote — not a question — and continue:

> **Version:** `v1.2.0` (minor) — proceeding without confirmation per
> /release contract. Reasoning: <one-line>. Override by re-invoking with
> an explicit version arg.

### Step 3 — Merge integration → main

If the current branch is `main` and pre-flight found nothing to merge,
**skip** (cutting on already-merged work is valid). Otherwise:

1. `git checkout main`
2. `git pull --ff-only origin main` — failure means main moved
   unexpectedly → hard-stop.
3. `git merge --no-ff <integration-branch> -m "Merge <branch> into main
   (<version>)"` — `--no-ff` preserves the integration history as a merge
   bubble.
4. Conflicts → hard-stop; report exactly which files; do not resolve.

The merge commit is the release commit. Capture `RELEASE_SHA=$(git
rev-parse HEAD)`.

### Step 4 — Tag the release commit (local)

Build the tag string. If `release-gate.md` (or a mounted pipelines
module) defines a tag-env helper, use it to produce `vX.Y.Z-<sha>-prod`;
otherwise use the plain semver `vX.Y.Z`. Create the **annotated** tag
locally (do not push yet):

```sh
git tag -a "<TAG>" -m "<release notes — format from release-rules.md>"
```

Pull the item list from the integration branch's commit history or the
`releases/RELEASES.md` entry's scope. The tag is **local-only** here;
push happens in Step 6, only if deploy succeeds.

### Step 5 — Deploy

Run the deploy command (from `release-gate.md`) in the foreground;
capture full output. Success → Step 6. Failure → **hard-stop**, no retry;
report the exact command and error. The local tag from Step 4 remains;
the user decides whether to delete it or keep it as a record of the
attempt.

### Step 6 — Push the tag

After deploy success:

```sh
git push origin main
git push origin "<TAG>"
```

Two separate pushes. If the tag push fails (network, auth), report a
partial-state warning: the deploy went live but the tag isn't remote yet;
the user pushes it manually.

### Step 7 — Record the release

**Audit ledger** (if the project has one — `tasks/AUDIT.md` from the
tasks module, or the project's own `AUDIT.md`) — add a 🚀 entry under
today's date header:

```markdown
- 🚀 **Released v1.2.0** — <one-line summary>. Tag `<TAG>`.
  Integration: <branch>.
```

**`releases/RELEASES.md`** — three sub-steps per `release-rules.md`:

1. **Stamp the "🚧 Next" entry**: change `🚧` to `✅`, replace the
   `next release — accumulating since …` line with the real release
   theme, add `shipped <YYYY-MM-DD> · tag <tag> · sha <short-sha>` (two-
   space indented).
2. **Cross-check** the item list against commits since the last tag; add
   any missed ids (silently — caught at pre-flight), remove stale claims
   (none should exist).
3. **Create a new "🚧 Next" entry** above the stamped one, version =
   previous + default bump, item list `(nothing yet)`.

Commit both to `main` as the post-release audit commit (`git commit -m
"audit: record v1.2.0 release"` → `git push origin main`). Partial state
("deploy shipped but no record") is not acceptable; a failed commit is
reported as a partial-state warning.

### Step 8 — Closing report

Render the deploy completion report: a status box (pre-flight / merge /
tagged / deploy / tag pushed / recorded, each ●), then the tag, the
branch, who deployed, start/complete timestamps, the live URL (from
`release-gate.md` / `CLAUDE.md`), and the GitHub release-tag URL. Below
the box: **Shipped** (the item list), **Decisions made** (version +
deploy command, flagged ⚠️ since no confirmation was taken), and
**Rollback** (the project's rollback command + the note that rollback
reverts the build but leaves the tag in place).

If any step partially succeeded (deploy went through, tag/record failed),
use a WARNING block instead — a clean deployment box implies a clean
release, which a partial state isn't.

## What you must NOT do

- **Don't ask for confirmation at soft gates.** Invocation is consent. If
  you find yourself prompting "Proceed?" / "Deploy now?" / "Confirm
  version?" — stop.
- **Don't retry a failed step.** Hard-stop, report, let the user
  re-invoke. Partial deploy state is dangerous; investigate first.
- **Don't push a tag before deploy succeeds.** Tag local in Step 4,
  pushed in Step 6.
- **Don't run lightweight tags.** Annotated only.
- **Don't deploy with a dirty working tree.** Hard-stop at pre-flight.
- **Don't invent a deploy or test command.** If `release-gate.md` names
  none, hard-stop and ask the user to fill the gate. Guessing prod
  commands is the one inference this skill won't make.

## Edge cases

- **No annotated tags yet** (first release): bootstrap per the project's
  tagging rule (`v1.0.0`, or `v0.1.0` for pre-1.0). The bootstrap is the
  choice; no version arg required.
- **Hotfix branch** (`hotfix/HOTFIX-NNN-slug`): defer to the hotfix path
  in `release-rules.md` — patch bump default, fast-track, audit entry
  tagged 🔥. Same contract; invocation is consent.
- **Empty release-gate**: if `.claude/release-gate.md` is still the
  seeded default (no real checks, no deploy command), hard-stop and tell
  the user to fill it before shipping.

## When NOT to use this skill

- **Just verifying a build** → the pipelines module's `/build`.
- **Appending a task to the next release** → `/release-add`.
- **Viewing the release timeline** → `/releases`.
- **Reverting / rolling back** → use the project's rollback command
  directly; record it in the audit ledger by hand.

## What "done" looks like for a /release session

Pre-flight passed; the integration branch merged to main; an annotated
tag created on the release commit AND pushed; the live build deployed;
the audit ledger 🚀 entry appended (if a ledger exists); the
`releases/RELEASES.md` entry marked ✅ Shipped; both committed to main;
the closing report rendered with the tag URL, commit SHA, and the
rollback escape hatch. If any of those didn't happen, the release isn't
done — say so explicitly. Partial state is the worst state to leave
undocumented.

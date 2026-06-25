---
name: release-add
description: Append an item to the top "🚧 Next" entry of `releases/RELEASES.md`. Triggered automatically by a review-merge skill and by `/release` after a merge to `main`, or invoked by the user after a manual merge. Idempotent — re-running for the same item is a no-op. Supports a `--since-last-tag` bulk mode to catch up after manual merges. Triggered when work lands on main and needs to be recorded for the next release — e.g. "/release-add TASK-042", "/release-add HOTFIX-007", "/release-add --since-last-tag", "track this for the next release".
---

# /release-add — append an item to the next-release entry

Single-purpose skill: take a `TASK-NNN` / `HOTFIX-NNN` (or a freeform
description) that just landed on `main`, find the top "🚧 Next" entry in
`releases/RELEASES.md`, and add it. **Idempotent** — running twice for
the same id is a no-op.

Small skills do one thing well. This skill manages one append operation
on one file. It does not commit, does not bump versions, does not ship.
The authoritative format is in `.claude/release-rules.md` → "Release
tracking."

## Behavior contract

- **Operate on `releases/RELEASES.md`.** If the file doesn't exist,
  create it from the format in `release-rules.md` (one "🚧 Next" entry,
  no shipped entries below).
- **Find the active "🚧 Next" entry.** Exactly one should exist at the
  top. If zero or multiple are found, **stop** and surface the finding —
  the file is in an unexpected shape.
- **Idempotency by id.** Parse the ids already listed under "🚧 Next." If
  the item to add is already there, exit cleanly without modifying the
  file. Report "already tracked."
- **Validate the id (soft).** `TASK-NNN` / `HOTFIX-NNN` format when the
  tasks module is in play. If the id doesn't resolve to a spec file, flag
  a warning but proceed — it may be filed elsewhere, or a placeholder.
  Freeform descriptions (no tasks module) are accepted as-is.
- **Pull the title from the spec file when available.** If a `tasks/`
  spec exists (`completed/` after merge, else `active/`, else
  `backlog/`), read the first `# <title>` line and use it. If not
  findable, use the merge-commit subject (stripped of any `TASK-NNN —`
  prefix) and flag it.
- **Append, don't reorder.** The item list under "🚧 Next" is in merge
  order. Append at the bottom.
- **Never auto-commit.** The file change sits in the working tree; the
  user reviews `git diff` and commits it in their next commit.

## The `--since-last-tag` mode

For catching up after manual merges that bypassed the review/release
skills:

```bash
/release-add --since-last-tag
```

1. Find the last release tag (`git describe --tags --abbrev=0`).
2. List every commit on `main` since that tag (`git log <tag>..main
   --pretty=%s`).
3. Extract every `TASK-NNN` / `HOTFIX-NNN` from commit subjects or from
   modified paths under `tasks/completed/` or `tasks/active/`.
4. For each unique id, run the single-add operation (idempotent —
   already-tracked ids are skipped).
5. Report the count of newly-tracked items.

## Process

1. **Read `release-rules.md`** to confirm the RELEASES.md format.
2. **Resolve the input.**
   - Single id arg → that one item.
   - `--since-last-tag` → bulk mode, per above.
   - No arg → look at the most recent merge commit on `main`, extract the
     id from its subject. If none found, **stop** and ask the user for an
     explicit id — guessing is worse than asking here.
3. **Open `releases/RELEASES.md`.** If absent, create it with one "🚧
   Next" entry, version = `<last-tag-version> + default-bump-minor`.
   Otherwise locate the "🚧 Next" entry (exactly one).
4. **Check idempotency.** Scan the entry's bullets for the id. If
   present, exit cleanly with "already tracked."
5. **Resolve the title.** Read the task spec from `tasks/completed/<id>-*`
   (most likely post-merge), else `tasks/active/`, else `tasks/backlog/`.
   Extract the first H1. If no spec anywhere, use the merge-commit subject
   and flag as an assumption.
6. **Append the bullet** (`- TASK-NNN — <title>`), preserving the entry's
   other content.
7. **Render a one-line confirmation:** `→ Tracked TASK-NNN for v0.38.0
   (next release).` or `→ Already tracked TASK-NNN for v0.38.0.`
8. **Don't commit.**

For `--since-last-tag`, loop the single-add and render:

```
→ Tracked N new item(s) for v0.38.0 (next release):
  TASK-042, TASK-043, HOTFIX-007
→ Skipped M already-tracked item(s):
  TASK-040, TASK-041
```

## When NOT to use this skill

- **Shipping a release** → `/release` (it stamps the "🚧 Next" entry ✅
  Shipped and creates the next one; this skill only appends).
- **Tracking work in a phase / roadmap** → the tasks module's `/task`.
  `/release-add` is about *what's about to ship*, not *what's planned*.
- **Viewing the timeline** → `/releases`.

## What "done" looks like

A modified `releases/RELEASES.md` with the id appended to the "🚧 Next"
entry's bullet list, uncommitted. One short confirmation line. Re-running
for the same id exits cleanly with "already tracked" and changes nothing.

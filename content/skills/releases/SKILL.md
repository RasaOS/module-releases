---
name: releases
description: Read and render the release timeline from `releases/RELEASES.md` — the current "🚧 Next" accumulator and the shipped history below it. Read-only; renders a clean view, never mutates the tracker. Triggered when the user wants to see release state — e.g. "/releases", "what's in the next release?", "show the release history", "what shipped in v1.2.0?".
---

# /releases — view the release timeline

Read + render `releases/RELEASES.md`. The read-side companion to
`/release` (cut) and `/release-add` (append). This skill **never
mutates** the tracker — it parses and presents.

The authoritative format is in `.claude/release-rules.md` → "Release
tracking." If this skill and that file disagree about the format,
`release-rules.md` wins.

## Behavior contract

- **Read-only.** Never edit `releases/RELEASES.md`. If the user wants to
  add an item, hand off to `/release-add`; to ship, hand off to
  `/release`.
- **Read state first.** Always read the current `releases/RELEASES.md`
  before rendering. If it doesn't exist, say so and point at
  `/release-add` (which seeds it) — don't fabricate a timeline.
- **Default view: the next release + recent history.** Render the "🚧
  Next" entry in full, then the most recent N shipped entries (default
  5). On request, render the full history or a single version.

## Operations

### Operation 1 — Default render (no arg)

1. Parse the "🚧 Next" entry: version, theme line, item bullets.
2. Parse the shipped entries below (most recent first).
3. Render:

```
🚧  NEXT  ·  v0.38.0   (accumulating since v0.37.0)

    TASK-042 — fix login bug on iOS
    TASK-043 — add filter UI to the inbox
    HOTFIX-007 — patch RBAC bypass on /admin
    — 3 item(s) staged —

────────────────────────────────────────────────

✅  v0.37.0  ·  shipped 2026-05-20  ·  tag v0.37.0  ·  sha f77a843
    /sync-all — autonomous variant of /sync
    · TASK-040, TASK-041

✅  v0.36.0  ·  shipped 2026-05-20  ·  tag v0.36.0  ·  sha 8274f0f
    status overhaul — blocked added (BREAKING)
    · TASK-038, TASK-039
```

Close with a one-line summary: `Next: v0.38.0 — 3 staged. Last shipped:
v0.37.0 (2026-05-20).`

### Operation 2 — Single version (`/releases v0.37.0`)

Render that one entry in full: theme, ship metadata (date / tag / sha),
and the full item list with titles.

### Operation 3 — Full history (`/releases --all`)

Render every shipped entry, newest first, in the compact form above.

### Operation 4 — Drift check (`/releases --check`)

Light read-only sanity pass (does not fix anything — that's `/release`'s
job): verify exactly one "🚧 Next" entry exists at the top, shipped
entries are in descending version order, and each shipped entry has a
date / tag / sha detail line. Report any anomalies as a short list. For
the deeper commits-vs-entry cross-check, point at `/release` pre-flight.

## When NOT to use this skill

- **Adding an item** → `/release-add`.
- **Shipping** → `/release`.
- **Editing the tracker by hand** → just edit `releases/RELEASES.md`;
  this skill is a viewer.

## What "done" looks like

A clean rendered view of the release timeline in chat, and an unchanged
`releases/RELEASES.md` on disk.

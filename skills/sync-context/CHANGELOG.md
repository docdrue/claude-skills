# Changelog — sync-context

## 1.1.0

Fixed: `.claude/sync-history.md` produced a merge conflict on every
`dev → stage → master` promotion.

**Cause:** the `<!-- last-synced: <sha> <date> -->` marker lived on line 1 of the
committed log and new entries were prepended at the top. Every sync on every
branch rewrote that same top region to a different commit SHA, so 3-way merges
always collided there.

**Fix:**
- Marker moved out of the committed file into `.claude/.sync-state` (gitignored,
  per-checkout) — the volatile SHA is never merged.
- Log is now append-only at the bottom under a fixed header that never changes,
  so branches only ever touch the file's tail.
- `.gitattributes` gains `.claude/sync-history.md merge=union` so concurrent
  appends merge with no conflict markers.
- `.gitignore` gains `.claude/.sync-state`.

**Regression guard:** never reintroduce the top-of-file marker or top-insertion
of entries, and keep the log header byte-stable across branches — those are what
caused the guaranteed conflict. The compression step (Step 7) is the only
operation that rewrites existing lines; run it only on the long-lived
integration branch, never on a branch about to be promoted.

## 1.0.0

Initial release.

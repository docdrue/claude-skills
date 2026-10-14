---
name: sync-context
description: Syncs CLAUDE.md with current project state — detects what changed since last sync, updates relevant sections, and writes all sync history and markers to .claude/sync-history.md. CLAUDE.md stays clean and minimal — never modified by this skill except to correct genuinely stale technical info. Invoke at session start and whenever significant code changes are made.
version: 1.0.0
user-invocable: true
argument-hint: "[check|force]"
---

# sync-context

Keeps CLAUDE.md accurate without polluting it with history. All sync tracking lives in `.claude/sync-history.md`.

---

## One-time project setup

When this skill is first used in a project, check for the following. If either is missing, add it and tell the user.

**1. Hook in `.claude/settings.json`** — triggers the skill automatically every session:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo '{\"hookSpecificOutput\":{\"hookEventName\":\"SessionStart\",\"additionalContext\":\"Before doing anything else, invoke the sync-context skill to check whether CLAUDE.md needs updating based on commits since the last sync.\"}}'"
          }
        ]
      }
    ]
  }
}
```

Preserve any existing keys — only merge in the `SessionStart` hook if it isn't already there.

**2. Session protocol note at the top of `CLAUDE.md`:**

```markdown
# Session Protocol

- At the start of every session, run the sync-context skill to check for drift between this file and the actual codebase.
- After completing any feature or significant change, run sync-context again before ending the session so .claude/sync-history.md stays current.
```

Add this only if CLAUDE.md doesn't already have a session protocol section.

---

## Step 0 — Build project context (first run only)

Check whether `CLAUDE.md` exists and contains meaningful content (more than just the Session Protocol header).

**If it does NOT exist or is empty:**

1. Ask the user: "Do you want to provide any context about this project before I analyse the codebase? (e.g. what it does, target audience, tech choices, team conventions, anything Claude should know — or press Enter to skip):"
2. Analyse the codebase to infer the project's structure and purpose:
   - Read `README.md`, `package.json` / `pyproject.toml` / `go.mod` (whichever exists), any `.env.example`
   - Scan the top-level directory structure and `src/` for architecture patterns, major modules, entry points
   - Read any existing linter/formatter configs to infer conventions
   - Note key scripts from `package.json` / `Makefile` / `justfile`
3. Combine both sources into the initial CLAUDE.md content:
   - User-provided context always takes precedence — never contradict it with codebase inferences
   - Add inferred facts that the user did not already cover
4. Write CLAUDE.md with the following sections populated from the combined context (add only sections where you have real content — omit empty ones):
   - **Session Protocol** (always first — see one-time project setup above)
   - **Tech Stack** — languages, frameworks, key libraries and their versions
   - **Architecture** — how the project is structured, major modules, entry points
   - **Conventions** — coding style, naming patterns, test setup
   - **Key Constraints** — env vars, external services, hard limits
   - **Commands** — the most important scripts/commands to run
   - **Gotchas** — known sharp edges, workarounds, non-obvious invariants
5. Show the user the generated CLAUDE.md and ask: "Here's the initial CLAUDE.md I've written. Type any corrections or press Enter to accept:"
6. If the user provides corrections — apply them. If they press Enter — keep it as-is.
7. Proceed to Step 1.

**If CLAUDE.md already exists with content** — skip this step entirely and go straight to Step 1.

---

## Step 1 — Find the last sync point

Read `.claude/sync-history.md` and look for the most recent marker at the top of the file:

```
<!-- last-synced: <commit-sha> <YYYY-MM-DD> -->
```

- If the marker exists, capture `<commit-sha>` as `$LAST_SHA` and `<YYYY-MM-DD>` as `$LAST_DATE`.
- If the file doesn't exist or has no marker, this is the first run — set `$LAST_SHA` to empty.

## Step 2 — Diff against HEAD

```bash
# What commits landed since last sync
git log $LAST_SHA..HEAD --oneline

# Which files changed and how much
git diff $LAST_SHA..HEAD --stat
```

If `$LAST_SHA` is empty (first run), list all tracked files with `git ls-files` instead.

**If `git log` returns nothing and this is not a forced run:** report "Nothing changed since last sync ($LAST_DATE). CLAUDE.md and .claude/sync-history.md left untouched." and stop.

## Step 3 — Identify what matters

From the diff, determine which CLAUDE.md sections are affected:

| Section | Affected when... |
|---|---|
| **Tech Stack** | A dependency was added/removed/upgraded in package.json, Gemfile, pyproject.toml, go.mod, etc. |
| **Architecture** | A new top-level directory appeared, a major module was added/removed, routing/middleware/entry points changed |
| **Conventions** | A linter config, formatter config, naming pattern, or test setup changed |
| **Key Constraints** | An env variable, feature flag, external API contract, or hard limit changed |
| **Commands** | A script in package.json / Makefile / justfile was added, renamed, or removed |
| **Gotchas** | A workaround was added, a known bug was patched, or a subtle invariant introduced |

Only flag a section if the diff contains evidence of a change to it — never speculatively.

## Step 4 — Identify major changes worth logging

**Log these:**
- New or removed dependency
- New architecture pattern or top-level structural change
- New convention introduced (linting rule, naming rule, test pattern)
- Breaking change to shared code (API contract, exported interface, env var rename)
- Config or environment change (new required env var, changed port, new external service)

**Skip these (trivial):**
- Formatting or whitespace only
- Typo fix with no behavioral effect
- Minor refactor with no behavioral change
- Comment-only edits
- Lock file updates with no version changes

## Step 5 — Update CLAUDE.md (only if stale info found)

For each affected section:
1. Read the current section content.
2. If it contains **genuinely stale information** (wrong version, removed command, outdated constraint), correct it.
3. If it's merely incomplete, add the missing item.
4. If it's accurate, leave it alone — do not rewrite healthy content.

Never add sync markers, timestamps, or changelog entries to CLAUDE.md.

## Step 6 — Write to .claude/sync-history.md

Create `.claude/` if it doesn't exist. For each major change:

1. Infer reasoning from commit messages, code comments, and PR/issue references. If the "why" cannot be found, write exactly: `not documented in commit history` — never invent a reason.
2. Format each entry:

```
YYYY-MM-DD — <short title (≤60 chars)>
What changed: <one factual sentence>
Why: <reasoning from commits/comments, or "not documented in commit history">
Affects: <comma-separated CLAUDE.md sections updated>
```

3. Place new entries **below** the `<!-- last-synced -->` marker, **above** existing entries (newest first).
4. Update (or create) the marker at the very top of `.claude/sync-history.md`:

```
<!-- last-synced: <current HEAD sha> <YYYY-MM-DD> -->
```

Never delete or rewrite existing entries — only append or compress.

## Step 7 — Compress old entries if needed

If `.claude/sync-history.md` exceeds 30 entries, summarize all but the 20 most recent into:

```markdown
## Earlier history

Summarized N entries from <oldest-date> to <cutoff-date>:
- <one-line summary per major theme or time period>
```

Replace the raw old entries with this block. Never compress any of the 20 most recent.

---

## Output

After running, always report:
- How many commits were scanned
- Which CLAUDE.md sections (if any) were updated and what changed
- How many entries were written to `.claude/sync-history.md`
- Whether compression occurred

If nothing changed, say so explicitly. Never silently succeed.

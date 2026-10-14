---
name: git-workflow
description: Enforce consistent git conventions: commit messages, branch names, and PRs. Use when committing, branching, or opening a PR.
version: 1.0.0
user-invocable: true
argument-hint: "[commit|branch|pr]"
---

# Git Workflow

Consistent git hygiene across all projects.

## Branch naming

Format: `<type>/<short-description>`

| Type | When to use |
|------|-------------|
| `feat/` | New feature |
| `fix/` | Bug fix |
| `chore/` | Tooling, deps, config |
| `docs/` | Documentation only |
| `refactor/` | Code restructure, no behaviour change |
| `hotfix/` | Urgent production fix |

Examples:
- `feat/user-authentication`
- `fix/cart-total-rounding`
- `chore/upgrade-dependencies`

Always branch from `main` unless fixing a live incident (branch from the release tag).

## Commit messages

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <short summary>

[optional body]

[optional footer]
```

**Rules:**
- Subject line: imperative mood, lowercase, no period, ≤72 chars
- Body: explain *why*, not *what* — the diff already shows the what
- Reference issues in the footer: `Closes #123`

**Types:** `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `perf`, `ci`

**Good examples:**
```
feat(auth): add magic link login flow
fix(cart): correct rounding on fractional quantities
chore(deps): upgrade vite to 5.4
```

**Bad examples:**
```
fix stuff          ← too vague
Added new feature  ← past tense, capitalised
WIP                ← never commit WIP to a shared branch
```

## Pull requests

**Title:** same format as a commit message — `type(scope): summary`

**Body must include:**
1. **What** — one sentence on the change
2. **Why** — motivation or issue link
3. **Test plan** — how to verify it works
4. **Screenshots** — for any UI change

**Before opening:**
- [ ] Branch is up to date with `main`
- [ ] No console errors or lint warnings
- [ ] Self-reviewed the diff — no debug code, no commented-out blocks
- [ ] PR is focused; one concern per PR

**Merge strategy:**
- Use **squash merge** for feature branches (keeps `main` history clean)
- Use **merge commit** only for long-lived branches with meaningful history to preserve

## When invoked as `/git-workflow`

1. Detect what the user is trying to do (commit, branch, or PR).
2. Apply the relevant convention above.
3. If committing: draft the message and ask the user to confirm before running `git commit`.
4. If branching: suggest a name and run `git checkout -b <name>` after confirmation.
5. If opening a PR: draft title + body using the template above, then run `gh pr create`.

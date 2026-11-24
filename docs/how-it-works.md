# How easy commit works

## The 8 steps

| # | Step | What it does |
|---|------|--------------|
| 0 | Preflight | Confirms a git work tree, resolves the default branch (falls back when `origin/HEAD` is unset), fetches if pushing, stops on in-progress rebase/merge. |
| 1 | Inspect | Reads status, staged/unstaged diffs and untracked files. If nothing to commit, looks for unpushed commits when push flags were given. |
| 2 | Safety screen | Skips secrets and build junk. Never uses `--no-verify`. |
| 3 | Group | Groups changes by what the commit subject would be. Matches the repo's existing message style. |
| 3b | Branch | Creates `<type>/<slug>` when HEAD is detached or a PR is requested from the default branch. |
| 4 | Commit | Stages each chunk exactly (whole files, renames, or individual hunks) and commits. |
| 5 | Push | Pushes the branch. Never force-pushes. Stops on rejection. |
| 6-7 | PR / Merge | `gh pr create --base <target>`, then `gh pr merge` for `-m`. |

## Grouping rules

- **Same subject, same commit.** A feature, its tests and the docs describing it travel together.
- **Different subject, different commit.** A bug fix never hides inside a feature commit.
- **Refactors and renames stay separate** from behavior changes.
- **Lockfiles go with the dependency change** that caused them.
- **Binary files go with whatever references them;** alone, they get their own commit.
- **Order matters:** helpers, types and config land before the code that uses them.
- Don't over-split. A single-purpose diff is one commit.

## Splitting one file into two commits

When a file contains two unrelated edits (say a bug fix directly above a new function), a normal diff shows one hunk. The skill:

1. Generates a zero-context diff: `git diff -U0 <file>`.
2. Keeps the header plus only the wanted lines (hand-writing a partial patch if the edits are contiguous).
3. Applies it to the index only: `git apply --cached --recount --unidiff-zero <patch>`.
4. Commits, then regenerates the diff for the next chunk, because line numbers shift.

## Commit message style

The skill reads `git log -10 --oneline` and follows what the repo already does. Conventional Commits is used only when the history is empty, very short, or already conventional:

```text
<type>(<scope>): <imperative subject, <=72 chars>
```

Types: `feat`, `fix`, `refactor`, `perf`, `docs`, `test`, `build`, `ci`, `chore`, `style`.

## Test matrix

Every scenario ran end to end in a throwaway repo with a local bare remote. PR and merge calls went to a stub `gh` that logs its arguments, so no real GitHub was touched. Each scenario was run twice: once on the first draft of the skill and again after fixing the gaps the first run exposed.

| ID | Scenario | Expected | Result |
|----|----------|----------|--------|
| A | `-pr` while on `main`, adjacent edits in one file | New branch, 3-4 commits, push, PR with base `main` | Pass |
| B | `-m develop` from a feature branch | Push, PR with base `develop`, merge call | Pass |
| C | Clean repo; non-git directory | Nothing to commit; stops at preflight | Pass |
| D | Rename, delete, lockfile, binary, plain-style history | Repo style followed, rename kept, lockfile grouped | Pass |
| E | Remote is ahead of local | Commit ok, push rejected, stop with suggestion | Pass |
| G | Detached HEAD with `-p` | Branch created, branch pushed, `main` untouched | Pass |
| H | `-pr` with only pre-existing unpushed commits on `main` | Branch at HEAD pushed, PR from it, `main` untouched | Pass |

Findings from the first round were fed back into the skill: hunk splitting for adjacent edits, preserving renames across index resets, a preflight step, branch naming rules, divergence reporting, PR-number lookup, and Windows PATH and quoting notes.

## What was not tested

- Real GitHub PR creation and merge (stubbed `gh` only).
- Protected-branch rules, required reviews and auto-merge on a real repository.

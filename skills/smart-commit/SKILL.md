---
name: smart-commit
description: Smart commit - analyzes all uncommitted changes, splits them into logical chunks (by intent, not by file), commits each chunk with a proper commit message, and optionally pushes (-p), opens a PR (-pr), or merges it (-m). Use whenever the user writes "smart commit", "/smart-commit", "smart commit -p", "smart commit -pr develop", "smart commit -m", or asks to commit current changes in logical groups, split a messy working tree into clean commits, or commit-push-PR-merge in one go, even if they don't say "smart".
---

# Smart Commit

Turn a messy working tree into a clean series of focused commits, then optionally push / open a PR / merge it. Each commit should be one logical change a reviewer can understand alone and revert alone. That is the whole point; every rule below serves it.

This skill is a self-contained, mechanical workflow. Do not interrogate the user with clarifying questions first; the flags are the spec. Only stop when a rule below says to.

## Invocation

```
smart commit [-p] [-pr] [-m] [target-branch]
```

| Flag | Effect |
|------|--------|
| (none) | Group and commit only. Nothing leaves the machine. |
| `-p` | After committing, push the current branch (set upstream if missing). |
| `-pr` | Push, then open a PR. Implies `-p`. |
| `-m` | Push, open a PR if none exists, then merge it. Implies `-pr`. |
| `target-branch` | Trailing bare word after all flags. PR base / merge target. Default: the repo's default branch (see step 0). |

Flags may be combined in any order. The first non-flag token is the target branch. Parse once, up front, and state the resolved plan in one line, including any branch you will create (e.g. "commit -> push `feat/foo` (new) -> PR into `main`").

Running a flag is the user's explicit consent to that outward action (push, PR, merge). Do not re-ask. That consent does **not** extend to pulling, rebasing, force-pushing or resolving conflicts; those are never done unasked.

## Workflow

### 0. Preflight
- `git rev-parse --is-inside-work-tree`. Non-zero exit (or `false`) means not a work tree: say so and stop, before any other step.
- Resolve the default/target branch (only needed for `-pr`/`-m` without an explicit target). Try in order: `git symbolic-ref --short refs/remotes/origin/HEAD`, then `git ls-remote --symref origin HEAD`, then `git branch -r` (prefer `origin/main`, else `origin/master`). `origin/HEAD` is often unset, so a failure of the first command is normal, not an error. If the explicit target doesn't exist on the remote, report and stop; never create it.
- If a push is planned and `git remote` lists `origin`, run `git fetch -q origin` so divergence is visible early (fetch only; it changes no local branches). Still commit if already diverged; the push step handles it.
- Rebase/merge in progress or unresolved conflicts (`.git/rebase-merge`, `.git/rebase-apply`, `.git/MERGE_HEAD` exist, or `git diff --name-only --diff-filter=U` is non-empty): stop and report.

### 1. Inspect
Run `git status --porcelain=v1`, `git diff`, `git diff --cached`, and list untracked files. Read small text untracked files to judge them; for binaries (images, archives) judge by name and location instead.

If there is nothing to commit:
- No push flags: say "nothing to commit" and stop.
- With `-p`/`-pr`/`-m`: look for existing unpushed work: `git rev-parse --abbrev-ref @{u}` to see whether an upstream exists; if so `git log @{u}..HEAD --oneline`, else `git log origin/<target>..HEAD --oneline` (don't run both). Commits found: skip to step 5 and use them (PR title/body come from this log). None found and no open PR to merge: say so and stop.

### 2. Safety screen (before staging anything)
- Skip and warn about likely secrets: `.env*`, `*.pem`, `*.key`, `credentials*`, tokens or keys visible in the diff.
- Skip build output and junk: `node_modules/`, `dist/`, `__pycache__/`, `*.log`, OS files. If they are not gitignored, mention it and suggest adding them to `.gitignore` (don't add it unasked, don't commit them).
- Never use `--no-verify`. If a hook fails, fix the cause and retry.

### 3. Group into chunks
Group by purpose, not by directory. Ask for each change: "what would the commit subject be?" Changes with the same answer belong together. Typical chunks:
- a feature plus its tests and the docs describing it
- a bug fix
- a refactor/rename (keep separate from behavior changes)
- config, dependency bumps, formatting-only changes
- lockfiles go with the dependency change that caused them

Rules of thumb:
- Split only where subjects genuinely differ. A single-purpose diff is one commit; don't invent splits. Two tiny unrelated changes are fine as two commits, but several trivial same-flavor edits (e.g. a rename plus the delete of its old sibling) can share one.
- A binary file (image, font) goes with the code/docs that reference it; if nothing does, it is its own chunk.
- Order chunks so dependencies (types, helpers, config) land before their users, and the repo builds at each step where feasible.
- If one file mixes unrelated changes, split it by hunk (step 4). Otherwise keep the file whole.

#### Commit message style
Run `git log -10 --oneline` now. Follow the repo's existing style (capitalised imperative, ticket prefixes, whatever it uses). Only when the history is empty, has fewer than ~3 commits, or already uses Conventional Commits, use:
```
<type>(<optional scope>): <imperative subject, <=72 chars, no trailing period>

<optional body: why, not what; wrap at ~72>
```
Types: `feat`, `fix`, `refactor`, `perf`, `docs`, `test`, `build`, `ci`, `chore`, `style`.

Append the attribution trailer your environment specifies for commits (if any) to each message body.

Show the plan (chunk -> files -> message). One line if there is one chunk; a short list otherwise. Then proceed without waiting for approval; they asked for automation. Pause only if a chunk is genuinely ambiguous or a safety screen tripped.

### 3b. Branch (only when needed)
Create a branch before committing when: HEAD is detached; or `-pr`/`-m` was given while on the default/target branch (a PR from `main` into `main` is meaningless). Decide the chunk plan and messages first, then name the branch `<type>/<slug>`: type of the dominant chunk (largest; on a tie, the oldest/first), slug from its subject with the type prefix removed, lowercased, non-alphanumerics (including dots) turned into single `-`, filler words dropped, <=40 chars (shorten by keeping the first 3-4 meaningful words). Trivial or mixed changes fall back to `chore/<slug>`. The same rule applies when no new commits are made (step 5 case): use the existing unpushed commits as the chunks. Plain `-p` on a normal branch (even `main`) needs no new branch; push as asked.

### 4. Commit each chunk
For each chunk, in order:
1. Reset the index: `git reset -q` (does not touch the working tree).
2. Stage the chunk: `git add -- <paths>`.
   - **Renames/deletes:** a rename was staged as delete+add before the reset. Stage both the old and new path (`git add -- old new`), or git records an add and leaves the delete behind. Confirm with `git diff --cached --stat -M`.
   - **Hunk-level splits:** build a patch of only the wanted hunks and apply it to the index: `git apply --cached --recount --unidiff-zero <patch>`. Write the patch to a temp file outside the repo so it never appears as untracked.
     Adjacent changes (a fix right above a new function) collapse into one hunk in a normal `git diff`, which can't be split. Use `git diff -U0 <file>` instead: one hunk per contiguous edit. Keep the file header plus only the wanted hunks. If a `-U0` hunk still mixes concerns (contiguous `-`/`+` lines from two changes), hand-write a partial patch containing only the lines for this chunk and commit it first; the remainder stays in the working tree for the next chunk. `--recount` repairs the counts. Commit the chunk that is easiest to isolate first, the remainder goes in whole afterwards, as long as dependency order still holds.
     Regenerate the `-U0` diff for each chunk after the previous chunk is committed, because later hunk line numbers shift.
   - Verify with `git diff --cached --stat`.
3. Commit with the message via `-F <tempfile>` (outside the repo) or repeated `-m` args, so multi-line bodies survive Windows quoting.
4. Confirm with `git log -1 --stat` and move on.

After the last chunk, `git status` must be clean except for deliberately skipped files. Report leftovers.

### 5. Push (`-p`, `-pr`, `-m`)
- `git push -u origin HEAD` if no upstream; otherwise `git push`. This also covers a freshly created branch and a detached-HEAD run (push the new branch, never the target branch).
- Never force-push. If rejected as non-fast-forward: stop. Report the rejection, the divergent remote commit(s) (`git log HEAD..@{u} --oneline`), and suggest `git pull --rebase` for the user to run. Do not run it yourself.
- If unpushed commits already sit on the default branch and `-pr`/`-m` was given: create `<type>/<slug>` at HEAD with `git branch <name>` (no checkout), push that (`git push -u origin <name>`), and open the PR from it. Stay on the current branch, leave local `main` alone (don't reset it), and tell the user it is still ahead of the remote.

### 6. Pull request (`-pr`, `-m`)
- `gh` problems (not installed, not authenticated, or the remote isn't a GitHub host): commits and push still stand. Report that the PR step needs `gh auth login` or a GitHub remote, and stop.
- Check for an existing open PR: `gh pr list --head <branch> --state open`. If one exists, reuse it (the push already updated it).
- Otherwise: `gh pr create --base <target> --head <branch> --title "<title>" --body "<body>"`.
  - Title: the commit subject if one commit; else a short plain-language summary covering the commits (e.g. "Add one and two helpers").
  - Body: a short summary plus a bullet per commit with short hash (from `git log origin/<target>..HEAD --oneline`). Write it to a temp file outside the repo and pass `--body-file`, so multi-line text survives quoting. End with the PR attribution line your environment specifies, if any.
- Print the PR URL.

### 7. Merge (`-m`)
- PR number: from the URL `gh pr create` printed, or `gh pr view --json number -q .number`.
- `gh pr merge <number> --merge --delete-branch`; if the repo only allows another method (read the error), retry with `--squash` or `--rebase` and say which was used. Never use `--admin` to bypass required checks or reviews.
- Checks pending or reviews required: use `gh pr merge --auto` only if the repo supports it; otherwise report that merge is blocked and why, and stop. Don't poll in a loop.
- Don't switch branches, pull, or delete the local branch afterwards (`--delete-branch` only touches the remote branch when you're on it); just report the result.

## Final report
End with a compact summary: commits made (hash + subject), anything skipped and why, push status (include the rejection reason if it failed), PR URL, merge result. Keep it short.

## Edge cases
- **No remote** with a push flag: commit, then report that there is no remote.
- **Huge diffs (>~1500 lines):** group by file/path first, inspect hunks only where a file looks mixed.
- **Windows:** in Git Bash, PATH entries must be POSIX style (`/c/...`), not `C:/...`; use temp files outside the repo for patches and messages.

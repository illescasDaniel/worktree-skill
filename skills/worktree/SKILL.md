---
name: worktree
description: "Create, save, apply (merge back into the parent branch) and delete isolated git worktrees for agent tasks, setting up each new worktree's environment automatically. Use when asked to work in a new worktree, save a worktree's changes, apply or merge a worktree back, or delete one, and when a fresh linked worktree is missing its dependencies. Not for ordinary branch work in the main checkout."
license: MIT
metadata:
  author: Daniel Illescas Romero <illescas.daniel@protonmail.com>
  version: "1.0.0"
  tags: [git, worktree, branches, isolation]
---

# Worktree

## Purpose

Work on a task in an isolated checkout, land it on the branch it came from, then remove it.

## Usage

Invoked as `/worktree <create|save|apply|delete>`; without an argument (or in an agent with no slash arguments) infer
the mode from the request, and ask if it is unclear. Do only the mode asked for: Create never merges, Save never pushes,
Apply never deletes, Delete never merges.

First run `git rev-parse --show-toplevel` and `git worktree list` to see where you are. The **main checkout** is the
first entry of `git worktree list`; a **linked worktree** is any other.

## Create

1. Skip creation if you are already in a linked worktree (e.g. the app made it) and go to step 4.
2. Note the **parent branch**: `git branch --show-current`. If HEAD is detached, ask which branch this work belongs to.
3. Slug the task (lowercase, dashes, about 40 chars) and branch from the **current HEAD**, not `main`:
   `git worktree add -b <branch> <path>` with no start-point. Use the path/branch convention of the tool you run in
   (Claude Code: `.claude/worktrees/<slug>`, branch `worktree-<slug>`; Cursor: `~/.cursor/worktrees/<slug>`, branch
   `cursor/<slug>`). Record the parent so Apply knows where to merge:
   `git config branch.<branch>.worktreeParent <parent>`. In Claude Code, `EnterWorktree({path})` on the existing path
   switches the session into it.
4. **Set up the environment now, without waiting for the user to ask.** A worktree holds only tracked files, so
   untracked-but-required things (virtualenv, `node_modules`, build output, `.env` copies, generated code) are
   missing and the first test or build run will fail.
   - Find the project's own instruction first: a "worktree" / "setup" section in `AGENTS.md`, `CLAUDE.md` or
     `README.md`, or a script it names (for example a `copy-venv` script). Follow it exactly and run the setup script
     it names.
   - If nothing is documented, do the equivalent yourself: prefer copying from the main checkout when the lockfile is
     identical, otherwise reinstall from the lockfile (`uv sync`, `npm ci`, `pnpm i --frozen-lockfile`,
     `bundle install`, `swift package resolve`, ...). Python venvs hold absolute paths (shebangs, editable `.pth`), so a
     copied one must be fixed or rebuilt rather than trusted.
   - Verify: import the package from the worktree's own sources, run the test runner's `--version`, or the project's
     fastest check. A copy that still points at the main checkout is a failure.
   - Re-run this step whenever you enter a worktree whose dependencies look missing or stale.
5. Report the branch, the parent branch, the path and what was set up.

## Save

Commit the task's work on the worktree branch. Review `git status` and `git diff`, update any memory or notes file the
project requires, stage only the task's files (never secrets, never `git add .`, keep unrelated staged work as it was),
and commit with a why-focused message in the repo's style. Do not push unless the user asked; if the branch has no
upstream yet, that first push sets it (`-u origin HEAD`).

## Apply

1. Stop if there is no linked worktree. Read the parent: `git config branch.<branch>.worktreeParent`. If it is not
   recorded, ask which branch to merge into; never assume `main`, `master` or `develop`.
2. Save first if there is pending work (see Save). Never copy uncommitted diffs into the parent instead.
3. Merge in the checkout where the parent branch is checked out (find it in `git worktree list`; if it is checked out
   nowhere, ask): `git -C <that checkout> merge <worktree-branch>`, fast-forward when possible. Stop if there is nothing
   to merge.
4. Resolve conflicts yourself, keeping both sides' intent. Abort and ask only if two edits to the same logic cannot
   compose. Append to, never rewrite, any append-only decision logs the project keeps.
5. Move into the parent's checkout (`ExitWorktree({action: "keep"})` if you used `EnterWorktree`, otherwise use
   `git -C <checkout>` or `cd`).
6. Run the project's quality gate (from `AGENTS.md`/`CLAUDE.md`/`README.md`) there until it is green; fix failures
   there and commit them.
7. Report the SHAs, the parent branch and that the gate passed. Then **offer** to delete the worktree; do not delete or
   push on your own.

## Delete

1. Abort if the target is the main checkout. If the worktree has unmerged commits or uncommitted files, ask before
   discarding (suggest Save and Apply first).
2. Move out of it first. If you entered via `EnterWorktree`, `ExitWorktree({action: "remove"})` does everything
   (pass `discard_changes: true` only after the user confirmed).
3. Otherwise from the main checkout: `git worktree remove <path>` (`--force` only if the user asked to discard), then
   `git worktree prune`, then delete the generated branch (`git branch -d`, `-D` only if confirmed) and its
   `worktreeParent` entry (`git config --unset branch.<branch>.worktreeParent`; `git branch -d` normally drops it).
   Never delete the parent branch, `main`, `master`, `develop` or a long-lived branch.
4. An editor or terminal may keep the folder locked ("Permission denied", "directory not empty"). Success is: gone from
   `git worktree list` and the branch deleted. Report a leftover folder in one line; do not kill processes or retry.

## Examples

"Fix the login bug in a new worktree", in Claude Code from branch `feature/auth`:

```bash
git worktree add -b worktree-fix-login-bug .claude/worktrees/fix-login-bug        # Create
git config branch.worktree-fix-login-bug.worktreeParent feature/auth
# ...set up the environment, work, then Save: commit on worktree-fix-login-bug
git -C <checkout of feature/auth> merge worktree-fix-login-bug                    # Apply
git worktree remove .claude/worktrees/fix-login-bug && git worktree prune          # Delete, after the user agrees
git branch -d worktree-fix-login-bug
```

## Requirements

- `git` 2.22 or later (`git worktree`, `git branch --show-current`), run inside a repository.
- The project's own setup commands (package manager, lockfile) for the environment step.
- Optional: Claude Code's `EnterWorktree` / `ExitWorktree` tools; plain `git` and `cd` work everywhere else.

## Limitations

- A new worktree holds only tracked files; ignored and untracked files must be rebuilt or copied by the setup step.
- Git lets a branch be checked out in only one worktree, so Apply merges where the parent branch is checked out.
- The parent branch is stored in the repository's local git config, not in the branch, so other clones do not see it.
- Never pushes, opens pull requests or deletes anything on its own.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Tests or imports fail right after Create | Dependencies missing, or a copied venv still points at the main checkout | Re-run Create step 4 and verify from the worktree's own sources |
| `git worktree add` says the branch already exists or is checked out | An earlier worktree used the same slug | Reuse that worktree, or ask before choosing a new slug |
| Apply cannot find the parent branch | `worktreeParent` not recorded, or the parent is checked out nowhere | Ask which branch and checkout to use; never assume `main` |
| `git branch -d` says "not fully merged" | The worktree branch has commits the parent lacks | Suggest Apply first; use `-D` only after the user confirms |
| "Permission denied" or "directory not empty" on Delete | An editor or terminal still holds the folder | Report the leftover folder; do not kill processes |

## Rules

- Create, save, apply and delete only when the user asked for that mode.
- Always set up the environment of a new worktree yourself, from the project's documented instructions.
- Never force-push, bypass hooks (`--no-verify`), use interactive (`-i`) flags, stage secrets, or change git config
  other than the `worktreeParent` key above.
- Never amend unless the user asked and HEAD is yours and unpushed.
- Do not push unless asked.

---
name: worktree
description: Create, apply and delete isolated git worktrees for agent tasks, and set up each new worktree's environment (venv, node_modules, build caches) automatically. Use when asked to start a task in a new worktree, "apply the worktree", merge a worktree back into the main checkout, "delete the worktree", or "save changes" (commit and push); also when you find yourself in a fresh linked worktree whose dependencies are missing.
---

# Worktree

Work on a task in an isolated checkout, land it on the parent branch, then remove it. The one thing that always goes
wrong is a fresh worktree with no working environment, so setting it up is part of creating it.

Detect where you are first: `git rev-parse --show-toplevel`, `git worktree list`. The **main checkout** is the first
entry of `git worktree list`; a **linked worktree** is any other. Pick the mode the user asked for.

## Create (and set up)

1. Skip creation if you are already in a linked worktree (e.g. the app made it) and go to step 3.
2. Slug the task (lowercase, dashes, about 40 chars). Branch from the **current HEAD**, not `main`:
   `git worktree add -b <branch> <path>` with no start-point. Use the path/branch convention of the tool you run in
   (Claude Code: `.claude/worktrees/<slug>`, branch `worktree-<slug>`; Cursor: `~/.cursor/worktrees/<slug>`, branch
   `cursor/<slug>`). In Claude Code, `EnterWorktree({path})` on the existing path switches the session into it.
3. **Set up the environment now, without waiting for the user to ask.** A worktree holds only tracked files, so
   untracked-but-required things (virtualenv, `node_modules`, build output, `.env` copies, generated code) are
   missing and the first test or build run will fail.
   - Find the project's own instruction first: a "worktree" / "setup" section in `AGENTS.md`, `CLAUDE.md` or
     `README.md`, or a script it names (for example a `copy-venv` script). Follow it exactly and run any script it
     names.
   - If nothing is documented, do the equivalent yourself: prefer copying from the main checkout when the lockfile is
     identical, otherwise reinstall from the lockfile (`uv sync`, `npm ci`, `pnpm i --frozen-lockfile`,
     `bundle install`, `swift package resolve`, ...). Python venvs hold absolute paths (shebangs, editable `.pth`), so a
     copied one must be fixed or rebuilt rather than trusted.
   - Verify: import the package from the worktree's own `src`, run the test runner's `--version`, or the project's
     fastest check. A copy that still points at the main checkout is a failure.
   - Re-run this step whenever you enter a worktree whose dependencies look missing or stale.
4. Report the branch, the path and what was set up.

## Apply (merge back into the main checkout)

1. Stop if there is no linked worktree. Note the worktree branch and the main checkout's branch.
2. Commit pending worktree work first (review `git status`/`git diff`, stage only task files, never secrets, match the
   repo's commit style). Never copy uncommitted diffs into main instead.
3. Merge from the main checkout: `git -C <main> merge <worktree-branch>` (fast-forward when possible). Stop if there is
   nothing to merge.
4. Resolve conflicts yourself, keeping both sides' intent. Abort and ask only if two edits to the same logic cannot
   compose. Append to, never rewrite, any append-only decision logs the project keeps.
5. Move into the main checkout (`ExitWorktree({action: "keep"})` if you used `EnterWorktree`, otherwise use
   `git -C <main>` or `cd`).
6. Run the project's quality gate (from `AGENTS.md`/`CLAUDE.md`/`README.md`) in the main checkout until it is green; fix
   failures there and commit them.
7. Report the SHAs, the branch and that the gate passed. Do not delete the worktree or push.

## Delete

1. Abort if the target is the main checkout. If the worktree has unmerged commits or uncommitted files, ask before
   discarding (suggest Apply first).
2. Move out of it first. If you entered via `EnterWorktree`, `ExitWorktree({action: "remove"})` does everything
   (pass `discard_changes: true` only after the user confirmed).
3. Otherwise from the main checkout: `git worktree remove <path>` (`--force` only if the user asked to discard), then
   `git worktree prune`, then delete the generated branch (`git branch -d`, `-D` only if confirmed). Never delete
   `main`, `master`, `develop` or a long-lived branch.
4. An editor or terminal may keep the folder locked ("Permission denied", "directory not empty"). Success is: gone from
   `git worktree list` and the branch deleted. Report a leftover folder in one line; do not kill processes or retry.

## Save changes

On "save changes": update any memory/notes file the project requires, review the diff, stage task files only, commit
(why-focused message in the repo's style), then `git push` (`-u origin HEAD` if no upstream). Only commit or push if
the user asked for it.

## Rules

- Create/apply/delete only when the user asks; Apply never deletes, Delete never merges.
- Always set up the environment of a new worktree yourself, from the project's documented instructions.
- Never force-push, bypass hooks (`--no-verify`), change git config, use interactive (`-i`) flags, or stage secrets.
- Never amend unless the user asked and HEAD is yours and unpushed.
- Do not push unless asked.

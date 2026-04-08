---
name: gs-insert
description: Use when the user wants to insert a branch at a specific position in an existing stack, not at the tip, or runs /gs-insert.
---

You are implementing `/gs-insert`. This inserts a new branch at a chosen position in an existing stack, stages and commits any pending changes, updates stack metadata, and rebases all downstream branches. No pushing, no PRs.

## Steps

### 1. Check working tree

```
git status --porcelain
```

If the output shows merge conflicts (lines starting with `UU`, `AA`, `DD`), stop and tell the user to resolve conflicts before inserting a branch.

Staged and unstaged changes are fine — they will be handled in step 7.

### 2. Discover the stack

**Follow the [stack discovery](../../docs/stack-discovery.md) pattern** (git config only, no fallback). Build the ordered branch list from root to tip.

Get the current branch:
```
git rev-parse --abbrev-ref HEAD
```

### 3. Show the stack with insertion positions

Display the full stack graph with numbered insertion points between each adjacent pair of branches. Mark the current branch with `(you are here)`. Example:

```
Stack:
  main
    ↓  ← [1] insert here
  stack/03-use-date-locale
    ↓  ← [2] insert here
  stack/04-locale-support  ← (you are here)
    ↓  ← [3] insert here
  stack/05-data-table-meta
```

Insertion position `[1]` means between `main` and the first stacked branch. The last position means between the last branch and the tip (appending to the end of the stack).

### 4. Ask for the insertion position

Ask:
> Where should the new branch be inserted? Enter a number from the list above.

Wait for the user to choose a number. Store:
- `<branch-above>` — the branch immediately above the insertion point
- `<branch-below>` — the branch immediately below (may be `null` if inserting at the tip)

### 5. Determine the new branch name

Read `CLAUDE.md` (if present) for branch naming conventions. Also look at recent branch names for patterns:
```
git branch --sort=-committerdate | head -20
```

Suggest a branch name consistent with the project convention. Ask the user to confirm or provide a different one. Wait for confirmation before proceeding.

### 6. Create the new branch from `<branch-above>`

Switch to the branch above the insertion point and create the new branch from its HEAD:
```
git checkout <branch-above>
git checkout -b <new-branch>
```

### 7. Handle staged/unstaged changes

**Follow the [safe staging flow](../../docs/staging.md) pattern.** Show the user what will be staged, warn on sensitive files, and stage by name. If there are no changes, ask whether to create an empty branch (skip step 8, continue with metadata update) or cancel; on cancel, undo with `git checkout <branch-above> && git branch -D <new-branch>`.

### 8. Suggest a commit message

Inspect the staged diff:
```
git diff --cached --stat
git diff --cached
```

Suggest a commit message following the project's conventions (Conventional Commits: `type(scope): message`). Check `CLAUDE.md` for any project-specific commit conventions.

Show the suggestion and ask the user to confirm or edit it. Wait for confirmation.

Once confirmed, commit:
```
git commit -m "<confirmed-message>"
```

Do NOT add `Co-Authored-By` lines. Do NOT add `🤖 Generated with Claude Code` lines.

### 9. Update stack metadata

Record the new branch's parent:
```
git config git-stack.<new-branch>.parent <branch-above>
```

If `<branch-below>` exists, update it to point to the new branch:
```
git config git-stack.<branch-below>.parent <new-branch>
```

### 10. Rebase downstream branches

Rebase each branch below the insertion point in order (closest first, down to the tip). For each downstream branch `<downstream>`, transplant it from its old parent to its new parent using `--onto`:

```
git rebase --onto <new-parent> <old-parent> <downstream>
```

Where `<new-parent>` is the branch immediately above it in the **updated** stack, and `<old-parent>` is the branch that was immediately above it in the **old** stack (before the insertion). For the first downstream branch, `<old-parent>` is the branch that was previously above the insertion point (i.e. `<branch-above>`), and `<new-parent>` is the newly inserted branch.

**Follow the [conflict handling](../../docs/conflict-handling.md) pattern.** Pause on genuine conflicts — list conflicted files, show the user, wait for resolution.

After all rebases, return to the new branch:
```
git checkout <new-branch>
```

### 11. Verify affected branches

Run the project's verification commands as defined in `CLAUDE.md` (e.g., `pnpm typecheck`, `pnpm lint`) on each affected branch (the new branch and all rebased downstream branches). For each branch, check it out before running verification:

```
git checkout <branch>
# run verification commands
```

Work through branches from the one closest to `main` to the tip. After verification, return to the new branch:

```
git checkout <new-branch>
```

If no verification commands are found in `CLAUDE.md`, skip this step.

Report any failures. Do not proceed past failures — ask the user to fix them first.

### 12. Report completion

Show a summary:
- New branch name and its position in the stack
- Commit hash and message (if a commit was made)
- List of rebased downstream branches
- Reminder that this is local only — run `/gs-submit` to push and open PRs, or `/gs-log` to see the full stack.

## Rules

- Do NOT push any branch.
- Do NOT create PRs.
- Do NOT add `Co-Authored-By` lines to commits.
- Do NOT add `🤖 Generated with Claude Code` lines.
- Follow the project's branch naming and commit message conventions (read from `CLAUDE.md` and existing patterns).
- Always confirm branch name and commit message with the user before acting.
- `gh` is not required — this is entirely local.

## Edge Cases

- **No stack exists:** Report error and suggest `/gs-create`.
- **Inserting at the tip (no branch below):** No downstream rebase needed. Only set `git-stack.<new-branch>.parent <branch-above>`.
- **Inserting at position [1] (between root and first stacked branch):** `<branch-above>` is `main` (or the stack root). `<branch-below>` is the first stacked branch — update its parent to point to the new branch, then rebase all downstream.
- **Rebase conflicts:** Stop, show the conflicting branch, instruct the user to resolve manually, and explain how to continue (`git rebase --continue`).
- **Dirty index with conflicts:** If `git status` shows merge conflicts before starting, stop and tell the user to resolve conflicts first.
- **Current branch is in the downstream:** After the rebase, the user's original current branch will have been rebased. Ensure `git checkout <new-branch>` lands correctly.

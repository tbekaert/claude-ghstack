---
name: gs-create
description: Use when the user wants to create a new stacked branch, add a branch on top of the current one, or runs /gs-create.
---

You are implementing `/gs-create`. This creates a new branch stacked on the current branch, stages and commits any pending changes, and records stack metadata in git config. No pushing, no PRs.

The user may invoke this as `/gs-create` or `/gs-create "feat: add retry logic"` (optional commit message argument).

## Steps

### 1. Discover the current stack position

**Follow the [stack discovery](../../docs/stack-discovery.md) pattern** (git config only, no fallback). Display the relevant portion of the stack so the user can see where they are. If the current branch is `main`, note that this will start a new stack.

Get the current branch name:
```
git rev-parse --abbrev-ref HEAD
```

Store it as `<current-branch>`.

### 2. Determine the new branch name

Read `CLAUDE.md` (if present) for branch naming conventions. Also look at recent branch names for patterns:
```
git branch --sort=-committerdate | head -20
```

Common patterns to look for: `type/ticket-slug`, `user/ticket-description`, `devin/ticket-id-slug`, etc.

If the user passed a commit message argument (e.g. `/gs-create "feat: add retry logic"`), derive a branch name suggestion from it.

Suggest a branch name and ask the user to confirm or provide a different one. Wait for confirmation before proceeding.

### 3. Create the new branch

Once the branch name is confirmed, create and switch to it from the current HEAD:
```
git checkout -b <new-branch>
```

### 4. Store parent metadata

Record the stack relationship in git config:
```
git config git-stack.<new-branch>.parent <current-branch>
```

### 5. Handle staged/unstaged changes

**Follow the [safe staging flow](../../docs/staging.md) pattern.** Show the user what will be staged, warn on sensitive files, and stage by name. If there are no changes, ask whether to create an empty branch or cancel; on cancel, undo with `git checkout <current-branch> && git branch -D <new-branch> && git config --unset git-stack.<new-branch>.parent`.

### 6. Suggest a commit message

If the user passed a commit message argument, use it as the suggestion. Otherwise, inspect the staged diff:
```
git diff --cached --stat
git diff --cached
```

Suggest a commit message following the project's conventions (Conventional Commits: `type(scope): message`). Check `CLAUDE.md` for any project-specific commit conventions.

Show the suggestion and ask the user to confirm or edit it. Wait for confirmation.

### 7. Commit

Once the message is confirmed, commit:
```
git commit -m "<confirmed-message>"
```

Do NOT add `Co-Authored-By` lines. Do NOT add `🤖 Generated with Claude Code` lines.

### 8. Report completion

Show a summary:
- New branch name
- Parent branch
- Commit hash and message
- Reminder that this is local only — run `/gs-submit` to push and open PRs, or `/gs-log` to see the stack.

## Edge Cases

- **Current branch is `main`:** This starts a new stack. Parent will be `main`. Proceed normally.
- **Current branch already has a child in the stack:** The new branch is stacked on top of the current branch (at the current HEAD), not inserted between the current branch and its existing child. The existing child's parent is not modified.
- **No changes to commit:** Ask the user whether to create an empty branch (branch + metadata only) or cancel. See step 5.
- **Dirty index with conflicts:** If `git status` shows merge conflicts, stop and tell the user to resolve conflicts before creating a stack branch.

## Rules

- Do NOT push any branch.
- Do NOT create PRs.
- Do NOT add `Co-Authored-By` lines to commits.
- Do NOT add `🤖 Generated with Claude Code` lines.
- Follow the project's branch naming and commit message conventions (read from `CLAUDE.md` and existing patterns).
- Always confirm branch name and commit message with the user before acting.

---
name: gs-sync
description: Use when the user wants to sync their stack after upstream changes, rebase the stack after amending commits, or runs /gs-sync.
---

You are implementing `/gs-sync`. This rebases the entire stack so each branch is layered correctly on the one below it.

## Steps

### 1. Discover the stack

**Follow the [stack discovery](../../docs/stack-discovery.md) pattern** (with PR chain fallback). Build the ordered branch list from root to tip.

Display the full stack so the user can see what will be rebased:

```
Stack to sync:
  <root-branch>
  └─ stack/01-feature-a
  └─ stack/02-feature-b
  └─ stack/03-feature-c
```

### 2. Fetch origin

```
git fetch origin
```

This ensures the rebase is based on the latest remote state.

### 3. Rebase the chain

Iterate through the stack from bottom to top (i.e., the branch closest to `<root-branch>` first, up to the tip).

For each branch:

1. Check out the branch:
   ```
   git checkout <branch>
   ```

2. Determine its parent from `git config git-stack.<branch>.parent`.

3. Rebase onto the parent. For the **bottom branch** (whose parent is `<root-branch>`), rebase onto `origin/<root-branch>`:
   ```
   git rebase origin/<root-branch>
   ```
   For all **subsequent branches**, rebase onto the already-rebased branch below them (the local branch, not its remote):
   ```
   git rebase <parent-branch>
   ```

4. Handle any conflicts as described in step 4 below before moving to the next branch.

### 4. Conflict handling

**Follow the [conflict handling](../../docs/conflict-handling.md) pattern.** After each `git rebase`, auto-skip squash artifacts and pause for genuine conflicts, waiting for the user to resolve them before continuing.

### 5. Verify

After all branches are rebased without errors, run the project's verification commands. Read `CLAUDE.md` to find what those are (e.g. `pnpm typecheck`, `pnpm lint`).

If verification fails, report the failure output and stop. Do not proceed to the push step.

### 6. Ask about pushing

For each branch in the stack, check if a remote tracking ref exists (after the earlier `git fetch origin`):
```
git rev-parse --verify --quiet origin/<branch>
```

If the command exits 0, the branch has been pushed before. If at least one branch has a remote ref, ask the user:

```
Rebase complete. Push updated branches to remote? (yes / no)
```

Wait for the user's response.

- **If yes:** For each branch that has a remote ref (where `git rev-parse --verify --quiet origin/<branch>` exited 0), push with:
  ```
  git push --force-with-lease origin <branch>
  ```
  If a push fails due to a lease mismatch (the remote was updated since the last fetch), report the failure for that branch and continue pushing the remaining ones. Do not retry automatically.

- **If no:** Skip pushing. Note in the summary that branches were rebased locally only.

If no branch has a remote configured, skip this step entirely.

### 7. Report summary

Show a clear summary of what happened:

```
Sync complete:
  stack/01-feature-a   rebased onto origin/<root-branch>   pushed ✓
  stack/02-feature-b   rebased onto stack/01       pushed ✓
  stack/03-feature-c   rebased onto stack/02       local only (not pushed)
```

Possible statuses per branch:
- `rebased onto <base>` — clean rebase
- `rebased onto <base> (N commits skipped)` — squash artifacts were skipped
- `pushed ✓` — force-pushed to remote
- `push failed` — lease mismatch or other push error
- `local only (not pushed)` — user declined or branch has no remote

## Rules

- Always use `--force-with-lease`, never bare `--force`.
- Never auto-resolve genuine conflicts — always show them to the user and wait.
- Skip already-merged (squash artifact) commits automatically without asking.
- Rebase order is always bottom-to-top: the branch closest to `<root-branch>` is rebased first.
- The bottom branch always rebases onto `origin/<root-branch>` (the fetched remote ref), not the local `<root-branch>`.
- All other branches rebase onto their local parent (which was just rebased in the previous step).
- Do not modify git-stack metadata during sync — this skill only rebases, it does not restructure the stack.

## Edge Cases

- **Single-branch stack:** Rebase that one branch onto `origin/<root-branch>` normally.
- **Already up to date:** If a branch is already up to date after rebase, git will report "nothing to do" — note this in the summary as `already up to date`.
- **Rebase produces empty commit (fully squashed):** This will trigger the already-merged path — use `git rebase --skip`.
- **User aborts mid-sync:** Run `git rebase --abort` on the current branch, then switch back to the branch the user was on before `/gs-sync` was invoked. Report which branches were successfully rebased and which were not.
- **No remote for any branch:** Skip the push step entirely. Report that the stack was synced locally.
- **Verification commands not found in CLAUDE.md:** Skip the verify step and note it in the summary.

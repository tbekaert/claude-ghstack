# Conflict Handling Pattern

Used during `git rebase` operations within a stack. After each rebase command, check the exit code:

- **Exit 0:** Rebase completed cleanly. Continue to the next branch.
- **Non-zero exit:** A conflict occurred. Determine which kind:

## Already-merged commits (squash artifacts)

When a commit was squash-merged (or merge-committed) upstream, its changes already exist in the base and git cannot apply the commit cleanly — but there is no real conflict, the code is already there.

**Detection:** Check conflicting files after the rebase stops:

```bash
git diff --name-only --diff-filter=U
```

If the conflicting files show only identical upstream changes (the incoming change exactly matches what is already in the base), this is a squash artifact. Skip it automatically:

```bash
git rebase --skip
```

Keep calling `git rebase --skip` (checking after each call) until the rebase either completes successfully or a genuine conflict is detected. Do not ask the user before skipping — this is mechanical cleanup.

**When this applies:** Squash strategy and merge-commit strategy both produce this situation. Rebase strategy replays commits individually, so downstream branches generally do not produce squash artifacts.

## Genuine conflicts

If conflicting files have real divergent changes that cannot be auto-resolved:

1. List conflicting files:
   ```bash
   git diff --name-only --diff-filter=U
   ```

2. Show the list to the user with a clear message:
   ```
   Conflict on <branch> while rebasing onto <parent>:
   The following files have conflicts that need manual resolution:
     - src/foo.ts
     - src/bar.ts

   Please resolve the conflicts in these files, then reply "resolved".
   ```

3. **Do not auto-resolve.** Wait for the user to reply.

4. Once the user confirms resolution, stage the resolved files:
   ```bash
   git add <file1> <file2> ...
   ```

5. Continue the rebase:
   ```bash
   git rebase --continue
   ```

6. If more conflicts appear, repeat from step 1. Continue until the rebase completes or the user aborts.

7. If the user wants to abort:
   ```bash
   git rebase --abort
   ```
   Then stop and report which branch failed and what state was left.

# Conflict Handling Pattern

Used during `git rebase` operations within a stack. After each rebase command, check the exit code:

- **Exit 0:** Rebase completed cleanly. Continue to the next branch.
- **Non-zero exit:** A conflict occurred. Determine which kind:

## Already-merged commits (squash artifacts)

When a commit was squash-merged (or merge-committed) upstream, its changes already exist in the base and git cannot apply the commit cleanly — but there is no real conflict, the code is already there.

**Detection:** After the rebase stops, check whether the conflict is a squash artifact by accepting the current (base) version and comparing:

```bash
# During a rebase, --theirs refers to the branch being rebased onto (the base),
# and --ours refers to the commit being replayed. So --theirs accepts the base version.
git checkout --theirs .
git add .

# Check if the result is identical to the base
git diff --cached --quiet
```

If `git diff --cached --quiet` exits 0 (no differences), the commit being replayed is entirely redundant — it is a squash artifact. Skip it:

```bash
git reset --hard
git rebase --skip
```

If `git diff --cached --quiet` exits non-zero, there are real changes in this commit beyond what exists in the base. Reset and treat it as a genuine conflict:

```bash
git reset --merge
```

Then follow the genuine conflict flow below.

Keep checking after each `git rebase --skip` until the rebase either completes successfully or a genuine conflict is detected. Do not ask the user before skipping squash artifacts — this is mechanical cleanup.

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

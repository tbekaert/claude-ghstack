---
name: gs-merge
description: Use when the user wants to merge stacked PRs into main, land approved PRs from the stack, or runs /gs-merge.
---

You are implementing `/gs-merge`. This merges one or all PRs from a stacked PR chain into main with full cleanup.

## Steps

### 1. Check prerequisites

**Follow the [prerequisites check](../../docs/prerequisites.md) pattern.** If `gh` is missing, unauthenticated, or the working tree is dirty, stop with the appropriate error message before proceeding.

### 2. Discover the stack

**Follow the [stack discovery](../../docs/stack-discovery.md) pattern** (with PR chain fallback). Build the ordered branch list from root to tip. If no stack is found, stop and report: "No stack found. Use `/gs-create` to start one, or provide branch names manually."

### 3. Present stack for validation

Show the discovered stack and ask for confirmation:

```
Stack detected (N PRs):
  1. stack/01-feature-a  (PR #42)  → base: main
  2. stack/02-feature-b  (PR #43)  → base: stack/01-feature-a
  3. stack/03-feature-c  (PR #44)  → base: stack/02-feature-b

Does this look correct? (yes / no)
```

If the user says no, ask them to provide the correct branch names in order (base to tip).

### 4. Ask scope

```
Merge all N PRs or just the next one?
  - all  — merge each PR in sequence (stops on any failure)
  - next — merge only PR #NNN, rebase the rest, then stop
```

Wait for the user's response.

### 5. Check available merge strategies

Query the repo settings:

```bash
gh api repos/{owner}/{repo} --jq '.allow_squash_merge, .allow_merge_commit, .allow_rebase_merge'
```

Parse the three boolean values. Build a list of available options from those that are `true`. Only offer available options to the user.

Ask the user to pick a strategy, recommending squash if available:

```
Merge strategy?
  1. Squash (recommended) — combines all commits into one
  2. Rebase — replays commits individually onto main
  3. Merge commit — creates a merge commit
```

(Only show lines for enabled strategies.)

Wait for the user's response.

### 6. Per-merge cycle

For each PR to merge, execute these steps in strict order. Complete the full cycle before moving to the next PR.

#### 6a. Check CI

```bash
gh pr checks <number>
```

All checks must have status `pass` or `skipping`. If any check is `fail` or `pending`, stop immediately:

> PR #N has failing/pending CI checks: [list check names and statuses]. Aborting.

Do not proceed to the next step.

#### 6b. Check review threads

Run the following GraphQL query:

```bash
gh api graphql -f query='
  query($owner: String!, $repo: String!, $number: Int!) {
    repository(owner: $owner, name: $repo) {
      pullRequest(number: $number) {
        reviewThreads(first: 100) {
          nodes { isResolved }
        }
      }
    }
  }
' -f owner="<owner>" -f repo="<repo>" -F number=<number>
```

Count threads where `isResolved` is `false`. If any exist, ask the user:

> PR #N has N unresolved review comment(s). Proceed anyway or abort? (proceed / abort)

Wait for the user's response. If the user says abort, stop and report. If the user says proceed, continue to the next step.

#### 6c. Merge

```bash
gh pr merge <number> --squash    # or --rebase or --merge based on chosen strategy
```

**CRITICAL: Never use `--delete-branch`.** The branch must remain until the next PR has been retargeted.

#### 6d. Retarget PRs and update stack metadata

This must happen before the merged branch is deleted, so GitHub can correctly update the diff.

**If scope is "all":** Only the immediate next PR needs retargeting (its base becomes `main`), since the merge cycle will process it next:

```bash
gh pr edit <next-pr-number> --base main
```

Update its stack metadata to reflect the new parent:
```bash
git config git-stack.<next-branch>.parent main
```

**If scope is "next" (merge only one PR):** The immediate next PR must be retargeted to `main`, AND its stack metadata must be updated. All further downstream branches keep their existing parent pointers (they still chain off each other correctly):

```bash
gh pr edit <next-pr-number> --base main
git config git-stack.<next-branch>.parent main
```

After the full merge cycle completes for the single PR (including rebase in step 6g), report to the user:
> Remaining stack has been rebased. Run `/gs-submit` to update the remaining PRs on GitHub.

If this was the last PR in the stack, skip this step.

#### 6e. Delete merged branch

```bash
git push origin --delete <merged-branch>
git branch -D <merged-branch>
git config --unset git-stack.<merged-branch>.parent
```

The `git branch -D` is local cleanup — if the branch does not exist locally, ignore the error. The `git config --unset` removes the stack metadata entry for this branch.

#### 6f. Fetch

```bash
git fetch origin
```

This updates `origin/main` to the post-merge state before rebasing.

#### 6g. Rebase remaining stack

For each remaining branch in the stack, in order from the branch closest to `main` outward to the tip:

- The **first remaining branch** rebases onto `origin/main`
- Each **subsequent branch** rebases onto the locally-rebased branch below it (not its remote)

For each branch:

1. Rebase:
   ```bash
   git checkout <branch>
   git rebase <target>
   ```

2. **Follow the [conflict handling](../../docs/conflict-handling.md) pattern.** Auto-skip squash artifacts (expected after squash and merge-commit strategies); pause and wait for the user on genuine conflicts. **Only proceed to the push step after the rebase completes successfully.**

3. Push only if the branch has a remote ref (check with `git rev-parse --verify --quiet origin/<branch>`):
   ```bash
   git push --force-with-lease origin <branch>
   ```
   Skip pushing branches that have never been published — those should go through `/gs-submit` for PR title/description approval.

### 7. Verify

After all rebases complete, run the project's verification commands as defined in `CLAUDE.md` (e.g., `pnpm typecheck`, `pnpm lint`) on the remaining branches. If no verification commands are found, skip this step.

If verification fails, report the failure output but continue to the summary — the merge and retarget have already been completed.

### 8. Report summary

After all cycles complete (or after stopping on a failure), show a summary table:

```
## gs-merge summary

| PR | Branch | Status |
|----|--------|--------|
| #175 | stack/03-use-date-locale | Merged ✓ |
| #176 | stack/04-locale-support | Merged ✓ |
| #177 | stack/05-data-table-meta | CI failing ✗ (stopped) |
| #178 | stack/06-sortable-list | Not attempted |

Merged: 2, Stopped at: PR #177, Remaining: 1
```

Possible statuses per row:
- `Merged ✓` — PR merged, branch deleted, downstream rebased
- `CI failing ✗ (stopped)` — CI check failed; halted here
- `Unresolved reviews ✗ (stopped)` — user chose to abort on unresolved threads
- `Not attempted` — would have been merged next but a prior stop prevented it

## Rules

- **Never** use `--delete-branch` during merge — branch must survive until the next PR is retargeted
- **Always** retarget the next PR's base to `main` before deleting the merged branch
- **Always** use `--force-with-lease` for all pushes, never bare `--force`
- **Always** present the stack and get user confirmation before any merge
- **Stop** on CI failure — do not skip and continue
- **Ask** on unresolved review threads — do not auto-proceed
- **Never** auto-resolve genuine rebase conflicts — pause and wait for the user
- Clean up `git-stack.*` config entries for every merged branch
- One PR at a time — complete the full cycle before starting the next

## Edge Cases

- **Single-branch stack:** Merge that one PR. No retargeting needed. Rebase step is a no-op.
- **Squash strategy + remaining branches:** Expect rebase conflicts on already-merged commits — skip them all automatically with `git rebase --skip`.
- **Rebase strategy + remaining branches:** Commits replay cleanly; no skipping needed.
- **Merge commit strategy + remaining branches:** Treat like squash — skip duplicate commits automatically.
- **User aborts during rebase conflict:** Run `git rebase --abort`, switch back to the original branch, report which PRs were merged and which were not.
- **Push rejected (lease mismatch):** Report the error, tell the user to run `/gs-sync` first, and stop. Do not retry automatically.
- **`gh pr edit --base` fails:** Report the error and stop. Do not delete the branch until retargeting succeeds.
- **Last PR in stack (no next PR):** Skip the retarget step (step 6d). Delete the branch normally.
- **No `git-stack.*` config and no open PRs:** Stop with: "No stack found. Use `/gs-create` to start one."

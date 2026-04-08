---
name: gs-submit
description: Use when the user wants to publish their stack to GitHub, push branches and create or update PRs, or runs /gs-submit.
---

You are implementing `/gs-submit`. This pushes every branch in the current stack to origin and creates or updates GitHub PRs accordingly.

## Steps

### 1. Check prerequisites

**Follow the [prerequisites check](../../docs/prerequisites.md) pattern.** If `gh` is missing, unauthenticated, or the working tree is dirty, stop with the appropriate error message before proceeding.

### 2. Discover the stack

**Follow the [stack discovery](../../docs/stack-discovery.md) pattern** (with PR chain fallback). Build the ordered branch list from root to tip.

### 3. Show the stack graph

Display the full stack before doing anything, so the user knows what will be submitted. Example:

```
Stack to submit:
  main
  └─ stack/01-feature-a       (no PR yet)
  └─ stack/02-feature-b       (no PR yet)
  └─ stack/03-feature-c       (no PR yet)
```

For branches that already have a PR, show the PR number and status:
```
  └─ stack/01-feature-a       PR #42 (open)
```

Check for existing PRs with:
```
gh pr list --head <branch> --json number,state,title,body --limit 1
```

Run this for each branch in the stack. A branch has an existing PR if the result is non-empty.

### 4. Process each branch (bottom to top)

Iterate through the stack from the branch closest to the root outward to the tip.

For each branch, determine its parent branch from `git config git-stack.<branch>.parent`.

---

#### Case A: No existing PR

**a. Push the branch:**
```
git push --force-with-lease origin <branch>
```

If the push fails (e.g. no upstream set yet), add `-u` to set the upstream:
```
git push --force-with-lease -u origin <branch>
```

**b. Generate PR title and description:**

Get the diff between the parent branch and this branch:
```
git diff <parent-branch>...<branch>
git log <parent-branch>..<branch> --oneline
```

Read `CLAUDE.md` (if present) for commit/PR conventions (e.g. Conventional Commits, ticket prefixes). Use those conventions when generating the title.

Generate:
- **Title:** A short, conventional title summarising the branch's changes. Follow the project's commit style (e.g. `feat(scope): what it does`). Do not exceed 72 characters.
- **Description:** A concise summary of what changed, written as markdown. No boilerplate phrases ("This PR…", "In this PR…"). No "Co-Authored-By" lines. No "Generated with Claude Code" lines.

**c. Show for approval:**

Display the proposed title and description and ask the user to confirm, edit, or skip this branch:

```
PR for stack/01-feature-a (base: main)
Title: feat(auth): add token refresh endpoint
---
Description:
Adds a `/auth/refresh` endpoint that issues a new access token from a valid
refresh token. Includes rate limiting and audit logging.
---
[C]onfirm / [E]dit / [S]kip this branch?
```

Wait for the user's response before proceeding.

**d. Create the PR:**
```
gh pr create --base <parent-branch> --head <branch> --title "<title>" --body "<description>"
```

Record the created PR URL/number in the running summary.

---

#### Case B: PR already exists

**a. Check for local changes vs remote:**
```
git diff <branch> origin/<branch>
```

If the diff is empty, the branch has no new local commits since the last push. Skip this branch and note "no changes" in the summary.

**b. Push the updated branch:**
```
git push --force-with-lease origin <branch>
```

**c. Regenerate title and description** from the updated diff (same process as Case A step b).

**d. Compare with current PR:**

Fetch the current PR title and body:
```
gh pr list --head <branch> --json number,title,body --limit 1
```

Compare the regenerated title and description with the current ones. If they are not meaningfully different (minor whitespace aside), skip the edit step silently and note "pushed, PR unchanged" in the summary.

If they differ meaningfully, show the proposed changes for user approval (same format as Case A step c, but show old vs new).

**e. Update the PR if approved:**
```
gh pr edit <number> --title "<title>" --body "<description>"
```

---

### 5. Report summary

After processing all branches, show a clear summary table:

```
Summary:
  stack/01-feature-a   PR #42 created  → https://github.com/…/pull/42
  stack/02-feature-b   PR #43 created  → https://github.com/…/pull/43
  stack/03-feature-c   skipped (no changes)
```

Possible statuses per branch:
- `PR #N created` — new PR was just opened
- `PR #N updated` — existing PR was pushed + description updated
- `pushed, PR unchanged` — branch was pushed but PR title/body were not changed
- `skipped (no changes)` — branch matched remote exactly, nothing to do
- `skipped (user request)` — user chose to skip during approval

## Rules

- Always use `--force-with-lease`, never bare `--force`.
- PR base is always the branch's `.parent` in the stack metadata, never hardcoded to `main` (unless the parent actually is `main`).
- Idempotent: running twice with no local changes must be a no-op (all branches skipped).
- Do NOT add `Co-Authored-By` lines anywhere.
- Do NOT add `🤖 Generated with Claude Code` lines to PR descriptions.
- Never push a branch without user approval of the PR title/description (for new PRs), or without detecting a meaningful diff (for updates).
- Never modify git config, branch metadata, or stack structure — this skill is publish-only.
- Always confirm new PR content with the user before calling `gh pr create`.
- Follow project PR/commit conventions as described in `CLAUDE.md`.

## Edge Cases

- **Single-branch stack:** Process that one branch normally.
- **Bottom branch's parent is main:** Base the PR on `main`. This is correct.
- **Remote branch does not exist yet:** `git push --force-with-lease -u origin <branch>` will create it.
- **Push rejected (non-fast-forward and lease mismatch):** Report the error, tell the user to run `/gs-sync` to rebase first, and stop processing further branches.
- **gh pr create fails (e.g. PR already exists but wasn't detected):** Catch the error, report it, and continue with remaining branches.
- **No CLAUDE.md found:** Fall back to Conventional Commits style for PR titles.

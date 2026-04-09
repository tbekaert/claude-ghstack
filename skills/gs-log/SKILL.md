---
name: gs-log
description: Use when the user wants to see the current stack status, view PR status for stacked branches, or runs /gs-log.
---

Display the stack graph with branch status and PR info. This is a read-only operation — no mutations, no questions.

**Note:** `gh` CLI is used for PR info. If unavailable, the skill degrades gracefully (see Edge Cases).

## Steps

### 1. Stack Discovery

**Follow the [stack discovery](../../docs/stack-discovery.md) pattern** (with PR chain fallback). Build the ordered branch list from root to tip. If no stack is found, output: "No stack found. Use `/gs-create` to start one, or ensure you're on a branch that belongs to a stack." then stop.

Note: during fallback, list all open PRs (not just those matching the current branch) to discover the full chain.

### 2. Determine Current Branch

```bash
git rev-parse --abbrev-ref HEAD
```

### 3. Gather Info Per Branch

For each branch in the stack (from the branch farthest from `<root-branch>` to the one closest):

**Commit count vs parent:**
```bash
git rev-list --count <parent>..<branch>
```

**PR number + URL** (if the branch has a remote):
```bash
gh pr list --head <branch> --state open --json number,url --jq '.[0]'
```

**CI check status** (only if a PR exists):
```bash
gh pr checks <number> --json state --jq '[.[].state] | if any(. == "FAILURE" or . == "failure" or . == "ERROR" or . == "error") then "failed" elif any(. == "PENDING" or . == "pending" or . == "IN_PROGRESS" or . == "in_progress") then "pending" else "success" end'
```

Note: `gh pr checks --json state` may return lowercase values (e.g. `"failure"`, `"pending"`, `"success"`) depending on the `gh` version. The jq filter checks both cases for compatibility.

**Unresolved review thread count** (only if a PR exists):

Run the following GraphQL query to get the repo owner and name first:
```bash
gh repo view --json owner,name --jq '{owner: .owner.login, name: .name}'
```

Then run the GraphQL query:
```bash
gh api graphql -F owner='<owner>' -F repo='<name>' -F number=<pr_number> -f query='
query($owner: String!, $repo: String!, $number: Int!) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $number) {
      reviewThreads(first: 100) {
        nodes { isResolved }
      }
    }
  }
}
' --jq '[.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved == false)] | length'
```

### 4. Determine PR Status Indicator

For each branch:

- **No PR (not published):** branch has no open PR → `not published`
- **CI pending:** any check in PENDING or IN_PROGRESS state → `⏳ CI running`
- **CI failed:** any check in FAILURE or ERROR state → `✗ CI failed`
- **Unresolved comments:** CI green but unresolved thread count > 0 → `💬 N comments`
- **Ready:** CI green and 0 unresolved threads → `✓ ready`

### 5. Display the Stack Graph

Render the stack from root (oldest ancestor) to tip (newest branch), using `↓` arrows between levels.

Format rules:
- Current branch: prefix with `●` and append `← you are here`
- Other branches: prefix with two spaces for alignment
- PR number rendered as a clickable markdown link: `[PR #N](url)`
- Commit count: `N commit` (singular) or `N commits` (plural)
- The root anchor (`<root-branch>`) has no PR info or commit count

**Example output:**

```
  <root-branch>
    ↓
  stack/03-use-date-locale ([PR #175](https://github.com/owner/repo/pull/175) ✓ ready) 1 commit
    ↓
  stack/04-locale-support ([PR #176](https://github.com/owner/repo/pull/176) ✓ ready) 1 commit
    ↓
● stack/05-data-table-meta ([PR #177](https://github.com/owner/repo/pull/177) ⏳ CI running) 2 commits  ← you are here
    ↓
  stack/06-sortable-list (not published) 1 commit
```

Once displayed, the skill is done. No further action, no questions asked.

## Rules

- Read-only — never modify branches, config, or PRs
- Never ask questions — display and done
- Gracefully degrade when `gh` is unavailable rather than erroring

## Edge Cases

- **Branch in git config no longer exists locally:** Skip it and show `(branch deleted)` in the graph.
- **PR was closed or merged:** Show `(PR #N closed)` or `(PR #N merged)` instead of the status indicator.
- **`gh` CLI not available:** Show the stack graph with branch names and commit counts only — skip PR status, show `(gh unavailable)` once at the top.
- **Current branch is not in the stack:** Show the full stack graph but without the `●` marker. Add a note: "Current branch is not part of this stack."

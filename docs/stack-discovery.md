# Stack Discovery Pattern

## Determine the root branch

Before building the branch list, read the stored root branch:

```bash
git config git-stack.root
```

If the command returns a value, store it as `<root-branch>`.

If no value is set (legacy stack or first discovery), walk the parent pointers to find the branch with no `git-stack.*.parent` entry — that is the root. Then persist it:

```bash
git config git-stack.root <detected-root>
```

Return `<root-branch>` alongside the ordered branch list for all callers. All skills must use `<root-branch>` instead of hardcoding `main`.

---

## Full version (with PR chain fallback)

Use this version in skills that have `gh` available (gs-sync, gs-merge, gs-submit, gs-log).

### 1. Read git config

```bash
git config --get-regexp "git-stack\."
```

This returns lines like `git-stack.<branch>.parent <parent-branch>`. Parse the output to build parent→child relationships and reconstruct the ordered branch list from `<root-branch>` to tip by following `.parent` pointers.

### 2. Fallback — walk PR chain

If no `git-stack.*` keys exist, auto-detect from GitHub:

```bash
# Get PR info for current branch
gh pr list --head <current-branch> --state open --json number,baseRefName,headRefName --jq '.[0]'

# Walk down to root (follow base refs until no further open PR exists)
gh pr list --head <base-branch> --state open --json number,baseRefName,headRefName --jq '.[0]'

# Walk up from root (find PRs stacked on top)
gh pr list --base <branch> --state open --json number,headRefName,baseRefName --jq '.[0]'
```

The walk terminates when a `baseRefName` has no open PR targeting it — that branch is the root. Store it as `<root-branch>` and persist it: `git config git-stack.root <root-branch>`.

If the fallback finds a stack, populate git config for future use:

```bash
git config git-stack.<branch>.parent <parent-branch>
```

Repeat for each branch/parent pair discovered.

**Remote-only branches:** If the fallback discovers a branch that exists on the remote (has an open PR) but does not exist locally, fetch it first:

```bash
git fetch origin <branch>
git checkout -b <branch> origin/<branch>
```

Then record its metadata as normal. If the fetch fails, skip that branch and note it in the output.

### 3. No stack found

If neither source yields a stack, stop and report:

> No stack found. Use `/gs-create` to start one.

---

## Simple version (git config only, no fallback)

Use this version in local-only skills that do not require `gh` (gs-create, gs-insert, gs-move).

Run:

```bash
git config --get-regexp "git-stack\."
```

Parse the output to build parent→child relationships and reconstruct the ordered branch list from root to tip by following `.parent` pointers.

If no `git-stack.*` keys exist, stop and report:

> No stack found. Use `/gs-create` to start one.

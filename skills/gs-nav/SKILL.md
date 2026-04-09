---
name: gs-nav
description: Use when the user wants to switch to another branch in the stack, navigate the stack, jump to a specific stack position, or runs /gs-nav.
---

You are implementing `/gs-nav`. This switches to another branch in the current stack. No mutations to branches, metadata, or PRs.

The user may invoke this as `/gs-nav` (interactive), `/gs-nav next`, `/gs-nav 2`, etc.

## Steps

### 1. Discover the stack

**Follow the [stack discovery](../../docs/stack-discovery.md) pattern** (git config only, no fallback). Build the ordered branch list from root to tip. If no stack is found, stop and report: "No stack found. Use `/gs-create` to start one."

Get the current branch:
```
git rev-parse --abbrev-ref HEAD
```

Determine the current branch's position in the stack (1-indexed, where 1 is closest to `<root-branch>`). If the current branch is `<root-branch>` or not in the stack, note this — the user can still navigate.

### 2. Parse the argument

The user may pass an argument after `/gs-nav`. Parse it as follows:

| Argument | Action |
|---|---|
| *(none)* | Interactive mode — show the stack and ask the user to pick (see step 3) |
| `next` or `n` | Move one branch farther from `<root-branch>` (the child branch) |
| `prev` or `p` | Move one branch closer to `<root-branch>` (the parent branch) |
| `first` or `f` | Jump to the branch closest to `<root-branch>` (first in the stack) |
| `last` or `l` | Jump to the branch farthest from `<root-branch>` (tip of the stack) |
| `root` or `r` | Check out `<root-branch>` (exit the stack) |
| A number (e.g. `3`) | Jump to the Nth branch in the stack (1-indexed, 1 = closest to `<root-branch>`) |

If the argument matches the actual root branch name (e.g. `main`, `master`), treat it the same as `root`.

If the argument doesn't match any of the above, report: "Unknown argument. Use `next`, `prev`, `first`, `last`, `root`, or a branch number."

### 3. Interactive mode (no argument)

Show the stack with numbered branches and the current position:

```
Stack:
  <root-branch>
    ↓
  [1] feat/auth
    ↓
  [2] feat/api         ← (you are here)
    ↓
  [3] feat/ui

Jump to: [number] / [n]ext / [p]rev / [f]irst / [l]ast / [r]oot
```

Wait for the user's response, then parse it using the same rules as step 2.

### 4. Validate the target

Before checking out, validate:

- **`next` from `<root-branch>`:** Jump to the first stacked branch (closest to `<root-branch>`). This is not an error — proceed to step 5.
- **`next` from the tip:** Report "Already at the last branch in the stack (farthest from `<root-branch>`). Nowhere to go." and stop.
- **`prev` from the first branch:** Report "Already the first branch in the stack (closest to `<root-branch>`). Use `/gs-nav root` to check out `<root-branch>`." and stop.
- **`prev` from `<root-branch>`:** Report "Already on `<root-branch>`." and stop.
- **Number out of range:** Report "Branch number N is out of range. The stack has N branches." and stop.
- **Already on the target branch:** Report "Already on [branch name]." and stop.

### 5. Check out the target branch

```
git checkout <target-branch>
```

If the checkout fails (e.g., uncommitted changes conflict with the target branch), report the git error to the user. Do not add a blanket dirty-tree pre-check — let git handle it naturally.

### 6. Confirm

Show a one-line confirmation with position context:

```
Switched to feat/ui (3 of 3, last in stack)
```

Position labels:
- Branch 1: `(1 of N, first in stack)`
- Branch N (last): `(N of N, last in stack)`
- Any other: `(M of N)`
- `<root-branch>`: `(<root-branch>, outside stack)`

The skill is done. No further action, no follow-up questions.

## Rules

- Read-only — never modify branches, metadata, or PRs
- Never add a blanket dirty-tree pre-check — let `git checkout` fail naturally if there are conflicts
- `gh` is not required — this is entirely local
- In interactive mode, wait for the user to choose before checking out
- Use "closer to `<root-branch>`" / "farther from `<root-branch>`" for direction — never "top" / "bottom"

## Edge Cases

- **Current branch is `<root-branch>`:** The user is outside the stack. `next` jumps to the first stacked branch. `prev` reports "Already on `<root-branch>`." Interactive mode still shows the full stack.
- **Current branch is not part of the discovered stack:** A stack exists (git config has `git-stack.*` keys) but the current branch isn't in it. Report "Current branch is not part of this stack." and show the stack so the user can pick a branch to navigate to.
- **Stack has only one branch:** `next` and `prev` both report they're at the edge. `first` and `last` both go to the same branch. Number `1` works.
- **Checkout fails due to uncommitted changes:** Report the git error and suggest: "Commit or stash your changes first, or use `/gs-create` to start a new branch with your current changes."
- **Argument is a branch name instead of a number:** Not supported — report the unknown argument message. The user should use the number or interactive mode.

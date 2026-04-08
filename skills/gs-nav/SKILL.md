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

Determine the current branch's position in the stack (1-indexed, where 1 is closest to `main`). If the current branch is `main` or not in the stack, note this — the user can still navigate.

### 2. Parse the argument

The user may pass an argument after `/gs-nav`. Parse it as follows:

| Argument | Action |
|---|---|
| *(none)* | Interactive mode — show the stack and ask the user to pick (see step 3) |
| `next` or `n` | Move one branch farther from `main` (the child branch) |
| `prev` or `p` | Move one branch closer to `main` (the parent branch) |
| `first` or `f` | Jump to the branch closest to `main` (first in the stack) |
| `last` or `l` | Jump to the branch farthest from `main` (tip of the stack) |
| `main` | Check out `main` (exit the stack) |
| A number (e.g. `3`) | Jump to the Nth branch in the stack (1-indexed, 1 = closest to `main`) |

If the argument doesn't match any of the above, report: "Unknown argument. Use `next`, `prev`, `first`, `last`, `main`, or a branch number."

### 3. Interactive mode (no argument)

Show the stack with numbered branches and the current position:

```
Stack:
  main
    ↓
  [1] feat/auth
    ↓
  [2] feat/api         ← (you are here)
    ↓
  [3] feat/ui

Jump to: [number] / [n]ext / [p]rev / [f]irst / [l]ast / [main]
```

Wait for the user's response, then parse it using the same rules as step 2.

### 4. Validate the target

Before checking out, validate:

- **`next` from `main`:** Jump to the first stacked branch (closest to `main`). This is not an error — proceed to step 5.
- **`next` from the tip:** Report "Already at the last branch in the stack (farthest from main). Nowhere to go." and stop.
- **`prev` from the first branch:** Report "Already the first branch in the stack (closest to main). Use `/gs-nav main` to check out main." and stop.
- **`prev` from `main`:** Report "Already on main." and stop.
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
- `main`: `(main, outside stack)`

The skill is done. No further action, no follow-up questions.

## Rules

- Read-only — never modify branches, metadata, or PRs
- Never add a blanket dirty-tree pre-check — let `git checkout` fail naturally if there are conflicts
- `gh` is not required — this is entirely local
- In interactive mode, wait for the user to choose before checking out
- Use "closer to `main`" / "farther from `main`" for direction — never "top" / "bottom"

## Edge Cases

- **Current branch is `main`:** The user is outside the stack. `next` jumps to the first stacked branch. `prev` reports "Already on main." Interactive mode still shows the full stack.
- **Current branch is not part of the discovered stack:** A stack exists (git config has `git-stack.*` keys) but the current branch isn't in it. Report "Current branch is not part of this stack." and show the stack so the user can pick a branch to navigate to.
- **Stack has only one branch:** `next` and `prev` both report they're at the edge. `first` and `last` both go to the same branch. Number `1` works.
- **Checkout fails due to uncommitted changes:** Report the git error and suggest: "Commit or stash your changes first, or use `/gs-create` to start a new branch with your current changes."
- **Argument is a branch name instead of a number:** Not supported — report the unknown argument message. The user should use the number or interactive mode.

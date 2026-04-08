---
name: gs-move
description: Use when the user wants to change a branch's position in the stack, remove it from the stack, or runs /gs-move.
---

You are implementing `/gs-move`. This reorders a branch within the stack, or detaches it entirely. No pushing, no PRs.

## Steps

### 1. Discover the stack

**Follow the [stack discovery](../../docs/stack-discovery.md) pattern** (git config only, no fallback). Build the ordered branch list from root to tip.

Get the current branch:
```
git rev-parse --abbrev-ref HEAD
```

### 2. Show the stack graph

Display the full stack with the current branch marked. Example:

```
Stack:
  main
  └─ stack/01-feature-a
  └─ stack/02-feature-b  ← (you are here)
  └─ stack/03-feature-c
```

### 3. Ask what action to take

Present the following options:

> What would you like to do?
> 1. Move up — swap with the branch above (closer to main)
> 2. Move down — swap with the branch below (further from main)
> 3. Move to position — pick a specific position in the stack
> 4. Detach — remove from stack and make it a standalone branch off main

Wait for the user to choose. Then proceed to the matching section below.

---

## Action: Move Up

Swap the current branch with the one immediately above it.

**Precondition:** the current branch must not already be at the top of the stack (i.e. its parent cannot be the stack root such as `main`). If it is, report:
> Already at the top of the stack. Nothing to move.

**Example:**

Stack before: `main → A → B → C`. Current branch is `B`.

After "move up": `main → B → A → C`

Metadata updates:
```
git config git-stack.B.parent main
git config git-stack.A.parent B
git config git-stack.C.parent A
```

Rebase order:
1. `git checkout A && git rebase B`
2. `git checkout C && git rebase A`

Return to `B` when done.

**General algorithm:**

Let `current` = current branch, `above` = its parent, `above-parent` = parent of `above`, `below` = child of `current` (may be null).

1. Set `git-stack.current.parent = above-parent`
2. Set `git-stack.above.parent = current`
3. If `below` exists: set `git-stack.below.parent = above`
4. Rebase `above` onto `current`
5. If `below` exists: rebase `below` onto `above`
6. Continue rebasing any further downstream branches onto their new parents, in order.
7. Return to `current`.

---

## Action: Move Down

Swap the current branch with the one immediately below it.

**Precondition:** the current branch must not already be at the tip (bottom) of the stack. If it is, report:
> Already at the bottom of the stack. Nothing to move.

**Example:**

Stack before: `main → A → B → C`. Current branch is `B`.

After "move down": `main → A → C → B`

Metadata updates:
```
git config git-stack.C.parent A
git config git-stack.B.parent C
```

Rebase order:
1. `git checkout C && git rebase A`
2. `git checkout B && git rebase C`

Return to `B` when done.

**General algorithm:**

Let `current` = current branch, `parent` = parent of `current`, `below` = child of `current`, `below-child` = child of `below` (may be null).

1. Set `git-stack.below.parent = parent`
2. Set `git-stack.current.parent = below`
3. If `below-child` exists: set `git-stack.below-child.parent = current`
4. Rebase `below` onto `parent`
5. Rebase `current` onto `below`
6. If `below-child` exists: rebase `below-child` onto `current`
7. Continue rebasing any further downstream branches onto their new parents, in order.
8. Return to `current`.

---

## Action: Move to Position

Show the stack with numbered insertion positions (like `/gs-insert`). Example:

```
Stack:
  main
    ↓  ← [1] move here
  stack/01-feature-a
    ↓  ← [2] move here
  stack/02-feature-b  ← (you are here)
    ↓  ← [3] move here
  stack/03-feature-c
```

> Where should this branch be moved? Enter a number from the list above.

Wait for the user to choose a number. Skip the position that corresponds to the branch's current location (moving there is a no-op — report "Already in that position" and stop).

Once the target position is chosen, update metadata and rebase as needed:

1. Remove `current` from its current position: update the branch that was below `current` to point to `current`'s old parent.
2. Insert `current` at the target position: update `current`'s parent to `<branch-above-target>`, and update `<branch-below-target>.parent` to `current` (if a branch exists below).
3. Rebase all affected branches in top-down order. Affected branches are any branches whose parent changed or that sit downstream of a changed parent.
4. Return to `current`.

---

## Action: Detach

Remove the current branch from the stack entirely and make it a standalone branch off main.

**Example:**

Stack before: `main → A → B → C`. User detaches `B`.

Result: stack is `main → A → C`, `B` is standalone off main.

Metadata updates:
```
git config git-stack.C.parent A
git config --unset git-stack.B.parent
```

Rebase order:
1. `git checkout B && git rebase main` (detach onto main)
2. `git checkout C && git rebase A` (close the gap in the stack)

Return to `B` when done.

**General algorithm:**

Let `current` = current branch, `parent` = parent of `current`, `below` = child of `current` (may be null).

1. If `below` exists: set `git-stack.below.parent = parent`
2. Unset `git-stack.current.parent`:
   ```
   git config --unset git-stack.current.parent
   ```
3. Rebase `current` onto the stack root (e.g. `main`).
4. If `below` exists: rebase `below` onto `parent`, then continue rebasing any further downstream branches in order.
5. Return to `current`.

---

## Post-Move: Verify Affected Branches

After any move action, run the project's verification commands as defined in CLAUDE.md (e.g., `pnpm typecheck`, `pnpm lint`) on each affected branch (branches that were rebased or had metadata changed).

Report any failures. Do not declare the move complete until the user has acknowledged any failures.

## Conflict Handling

**Follow the [conflict handling](../../docs/conflict-handling.md) pattern.** Pause on genuine conflicts — list conflicted files, show the user, wait for resolution.

## Edge Cases

- **Stack has only one branch:** Moving up or down is not possible. Detach is allowed. Report appropriately.
- **Current branch is at the top (parent = main):** "Move up" is a no-op. Report and stop.
- **Current branch is at the tip:** "Move down" is a no-op. Report and stop.
- **Detaching the only branch in the stack:** Unset metadata and rebase onto main. The stack becomes empty.
- **Dirty working tree:** If `git status` shows uncommitted changes or unresolved conflicts before starting, stop and tell the user to clean up first.

## Rules

- Do NOT push any branch.
- Do NOT create PRs.
- Do NOT add `Co-Authored-By` lines.
- Do NOT add `🤖 Generated with Claude Code` lines.
- `gh` is not required — this is entirely local.
- Always confirm the chosen action with the user before making any changes.

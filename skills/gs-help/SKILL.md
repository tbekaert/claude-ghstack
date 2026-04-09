---
name: gs-help
description: Use when the user asks for help with stacked PRs, runs /gs-help, or wants to know what /gs-* commands are available.
---

Stacked PR Skills (claude-ghstack plugin)

  Local operations (no network):
    /gs-create   — Create a new branch stacked on current
    /gs-insert   — Insert a branch at a chosen position in the stack
    /gs-move     — Reorder or detach a branch in the stack
    /gs-nav      — Switch to another branch in the stack

  Network operations:
    /gs-submit   — Push all branches + create/update PRs
    /gs-sync     — Rebase the stack + optionally push
    /gs-merge    — Merge PRs into base branch with cleanup

  Info:
    /gs-log      — Show the stack graph with PR status
    /gs-help     — This help

  Typical workflow:
    1. /gs-create to build your stack locally
    2. /gs-submit to publish PRs
    3. /gs-sync after changes or base branch updates
    4. /gs-merge when PRs are approved

  Metadata is stored in .git/config (git-stack.* keys).
  No files to commit. Run /gs-log to see your stack.

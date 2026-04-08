# claude-ghstack

A Claude Code plugin for stacked PR management. Create, reorder, publish, sync, and merge stacked branches without leaving your editor.

## Commands

| Command | Network | Description |
|---|---|---|
| `/gs-create` | no | Create a new branch stacked on current |
| `/gs-insert` | no | Insert a branch at a chosen position in the stack |
| `/gs-move` | no | Reorder or detach a branch in the stack |
| `/gs-submit` | yes | Push all branches and create/update PRs on GitHub |
| `/gs-sync` | yes | Rebase the stack and optionally push |
| `/gs-merge` | yes | Merge approved PRs into main with cleanup |
| `/gs-log` | no | Show the stack graph with PR status |
| `/gs-help` | no | Show the reference card |

## How it works

Stack metadata is stored in `.git/config` under `git-stack.*` keys — no extra files to commit or track. Each skill reads the stack state, performs its operation, and updates the config keys.

Branches are linked in a chain: each branch records its parent. `/gs-submit` walks the chain bottom-up to create PRs with the correct base branch. `/gs-sync` rebases each layer in order after `main` moves.

## Installation

Copy (or symlink) this directory into your project's `.claude/plugins/` folder:

```bash
cp -r claude-ghstack /your/project/.claude/plugins/claude-ghstack
```

Then reload Claude Code. Run `/gs-help` to verify the plugin is active.

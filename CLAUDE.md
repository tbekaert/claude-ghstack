# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

claude-ghstack is a Claude Code plugin for stacked PR management. It provides nine slash-command skills (`/gs-*`) that create, reorder, publish, sync, and merge stacked branches without leaving the editor. There is no build step, no tests, and no runtime code — the repo is entirely markdown-based skill definitions.

## Development

There is no build, lint, or test step. All changes are to markdown files. To validate a skill, read it and check for internal consistency.

When bumping the version, update both `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`.

## Architecture

```
.claude-plugin/plugin.json        — Plugin manifest (name, version, metadata)
.claude-plugin/marketplace.json   — Marketplace definition for plugin distribution
skills/<name>/SKILL.md             — One skill per directory, each a self-contained prompt
docs/*.md                          — Shared patterns referenced by multiple skills
```

### Shared Patterns (docs/)

Skills reference these via relative links like `../../docs/stack-discovery.md`. If you modify a pattern, check which skills depend on it:

- **stack-discovery.md** — How to read `git-stack.*` config and optionally fall back to GitHub PR chains. Two variants: "simple" (local-only skills) and "full" (network skills with `gh` fallback).
- **prerequisites.md** — Pre-flight checks for network skills: `gh` installed, authenticated, clean working tree.
- **staging.md** — Safe staging flow: show changes, warn on sensitive files, stage by name (never `git add -A`). Handles already-staged files.
- **conflict-handling.md** — Rebase conflict protocol: auto-skip squash artifacts using `git checkout --theirs . && git diff --cached --quiet`, pause on genuine conflicts, wait for user.

### Stack Metadata

All stack state lives in `.git/config` under `git-stack.*` keys. No files are committed or tracked. Skills read/write this config directly via `git config`.

- `git-stack.root` — the base branch name (e.g., `main`, `master`). Detected via `gh` on first stack creation, read by all skills. Skills must never hardcode `main` — always use the value from `git config git-stack.root`.
- `git-stack.<branch>.parent` — the parent branch for each stacked branch.

### Skill Categories

- **Local-only** (no `gh` needed): `gs-create`, `gs-insert`, `gs-move`, `gs-nav` — use simple stack discovery (git config only). `gs-create`/`gs-insert`/`gs-move` require clean tree; `gs-nav` lets git handle checkout conflicts naturally.
- **Network** (`gh` required): `gs-submit`, `gs-sync`, `gs-merge` — use full stack discovery with PR chain fallback, delegate to prerequisites.md
- **Read-only**: `gs-log` (display only, no mutations), `gs-help` (static reference card)

## Skill Structure Conventions

Every skill follows the same structure:

1. **YAML frontmatter** — `name` and `description` (description starts with "Use when...")
2. **One-liner** — what this skill implements
3. **Steps** — numbered, sequential, with git commands in code blocks
4. **Rules** — hard constraints (always before Edge Cases)
5. **Edge Cases** — how to handle unusual situations (always after Rules)

When adding or editing skills:

- Use `git rebase --onto <new-parent> <old-parent> <branch>` when transplanting branches between parents (not plain `git rebase`)
- Check for remote refs with `git rev-parse --verify --quiet origin/<branch>` (not `git config --get branch.<branch>.remote`)
- Compare local vs remote with `git rev-parse` on both refs (not `git diff`)
- All local skills start with a dirty-tree pre-check (`git status --porcelain`)
- All mutating skills end with a verification step (read `CLAUDE.md` for project commands)
- Use "closer to `<root-branch>`" / "farther from `<root-branch>`" for direction — never "top" / "bottom"

## Key Conventions Across Skills

- Always use `--force-with-lease`, never bare `--force`
- Never add `Co-Authored-By` lines or "Generated with Claude Code" lines
- Always confirm branch names, commit messages, and PR descriptions with the user before acting
- Rebase order is always bottom-to-top (closest to `<root-branch>` first)
- Bottom branch rebases onto `origin/<root-branch>`; subsequent branches rebase onto their local parent
- PR base is always the branch's `.parent` in stack metadata — never hardcoded
- Never push branches that haven't been published yet — those must go through `/gs-submit` for PR approval

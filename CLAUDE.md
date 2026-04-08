# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

claude-ghstack is a Claude Code plugin for stacked PR management. It provides 8 slash-command skills (`/gs-*`) that create, reorder, publish, sync, and merge stacked branches without leaving the editor. There is no build step, no tests, and no runtime code — the repo is entirely markdown-based skill definitions.

## Architecture

```
.claude-plugin/plugin.json   — Plugin manifest (name, version, metadata)
skills/<name>/SKILL.md        — One skill per directory, each a self-contained prompt
docs/*.md                     — Shared patterns referenced by multiple skills
```

### Shared Patterns (docs/)

Skills reference these via relative links like `../../docs/stack-discovery.md`. If you modify a pattern, check which skills depend on it:

- **stack-discovery.md** — How to read `git-stack.*` config and optionally fall back to GitHub PR chains. Two variants: "simple" (local-only skills) and "full" (network skills with `gh` fallback).
- **prerequisites.md** — Pre-flight checks for network skills: `gh` installed, authenticated, clean working tree.
- **staging.md** — Safe staging flow: show changes, warn on sensitive files, stage by name (never `git add -A`).
- **conflict-handling.md** — Rebase conflict protocol: auto-skip squash artifacts, pause on genuine conflicts, wait for user.

### Stack Metadata

All stack state lives in `.git/config` under `git-stack.<branch>.parent` keys. No files are committed or tracked. Skills read/write this config directly via `git config`.

### Skill Categories

- **Local-only** (no `gh` needed): `gs-create`, `gs-insert`, `gs-move` — use simple stack discovery (git config only)
- **Network** (`gh` required): `gs-submit`, `gs-sync`, `gs-merge` — use full stack discovery with PR chain fallback
- **Read-only**: `gs-log` (display only, no mutations), `gs-help` (static reference card)

## Key Conventions Across Skills

- Always use `--force-with-lease`, never bare `--force`
- Never add `Co-Authored-By` lines or "Generated with Claude Code" lines
- Always confirm branch names, commit messages, and PR descriptions with the user before acting
- Rebase order is always bottom-to-top (closest to `main` first)
- Bottom branch rebases onto `origin/main`; subsequent branches rebase onto their local parent
- PR base is always the branch's `.parent` in stack metadata, not hardcoded to `main`

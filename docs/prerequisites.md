# Prerequisites Check Pattern

Run the following checks in order before performing any remote operations. Stop and report clearly if any fail.

```bash
gh --version
gh auth status
git status --porcelain
```

- If `gh` is not installed: report "The `gh` CLI is required. Install it from https://cli.github.com/"
- If `gh auth status` fails: report "`gh` is not authenticated. Run `gh auth login` first."
- If `git status --porcelain` returns output: report "Working tree is dirty. Commit or stash all changes before proceeding."

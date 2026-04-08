# Safe Staging Flow Pattern

Used before committing in skills that create or insert branches.

### 1. Check for changes

```bash
git status --short
```

### 2. Handle changes

- **If there are staged or unstaged changes** (including untracked files the user likely wants included):
  1. Show the user the `git status` output so they can see what will be staged.
  2. Warn if any potentially sensitive files (`.env`, credentials files, private keys) appear in the changeset — ask the user to confirm before staging them.
  3. Stage specific files by name rather than using a blanket `git add -A`:
     ```bash
     git add <file1> <file2> ...
     ```

- **If there are no changes at all:** Ask the user whether to create an empty branch (branch + metadata only, no commit) or cancel.
  - If empty branch: skip staging and commit, continue with metadata.
  - If cancel: undo the branch and stop.

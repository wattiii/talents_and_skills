---
name: git-workflow
description: Common git workflows and automation patterns for individuals and teams
version: 1.0.0
author: James Watt
license: MIT
platforms: [macos, linux]
metadata:
  hermes:
    tags: [git, workflow, version-control]
    category: productivity
---

# Git Workflow Skill

## When to Use
- Automating common git operations (branch management, PR creation, merge conflicts)
- Standardizing team git practices
- Reducing repetitive git command typing

## Procedure

### Branch Management
```bash
# Create and switch to a new branch
git checkout -b feature/new-feature

# Push branch and set upstream
git push -u origin feature/new-feature

# Return to main and pull latest
git checkout main
git pull origin main

# Delete merged branches
git branch --merged main | grep -v "\*\|main\|dev" | xargs -n1 git branch -d
```

### Pull Request Workflow
```bash
# Start PR workflow
git checkout -b feature/awesome-feature
# ... make commits ...
# Push and open PR
gh pr create --title "Awesome Feature" --body "Description of changes"

# After review, merge and clean up
git checkout main
git pull origin main
git branch -d feature/awesome-feature
```

### Conflict Resolution
```bash
# When merge conflict occurs
git status
# Open conflicted files in editor
git add <resolved-file>
git rebase --continue
# Or for merge:
git merge --abort
git merge feature/feature-branch
```

## Pitfalls
- **Force pushing to shared branches** — avoids rewriting history for teammates
- **Not pulling before pushing** — always pull main before pushing feature branches
- **Commit messages too vague** — use imperative mood: "Fix bug" not "Fixed bug"

## Verification
```bash
# Test branch creation
git checkout -B test-branch && git branch

# Test push setup
git push -u origin test-branch

# Verify upstream is set
git branch -vv
```
Expected: test-branch shows `[ahead N, behind N]` with upstream tracked.

# Common Commands Reference
| Command | Purpose |
|---|---|
| `git checkout -b <name>` | Create and switch to new branch |
| `git push -u origin <name>` | Push and set upstream |
| `git pull --rebase` | Update branch cleanly |
| `git request-pull` | Generate pull request summary |
| `git bisect` | Binary search for regressions |
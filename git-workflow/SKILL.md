---
name: git-workflow
description: Git branching strategies, commit conventions, and workflow patterns. Use when setting up Git workflows, writing commit messages, or managing branches and releases.
allowed-tools: Read, Grep, Glob, Bash
---

# Git Workflow Patterns

Quick reference for Git branching strategies and conventions.

## Branching Strategies

### GitHub Flow (Simple)
```
main ────●────●────●────●────●
         │    │    │    │    │
feature  └────┘    └────┘    └────┘
```
- Single `main` branch always deployable
- Short-lived feature branches
- Deploy after every merge
- Best for: continuous deployment, small teams

### Git Flow (Structured)
```
main ─────●─────────────────●
          │                 │
release   └──●──●──────────●┘
             │  │
develop ─●───●──●──●──●──●──●
         │      │     │
feature  └──────┘     └──────
```
- `main` for releases, `develop` for integration
- Feature branches from `develop`
- Release branches for stabilization
- Best for: scheduled releases, larger teams

### Trunk-Based (Fast)
```
main ────●────●────●────●────●────●
         │              │
feature  └──────────────┘
(very short-lived, < 1 day)
```
- Everyone commits to main (or very short branches)
- Feature flags for incomplete work
- Best for: experienced teams, CI/CD maturity

## Branch Naming

```
feature/TICKET-123-add-user-auth
bugfix/TICKET-456-fix-login-error
hotfix/TICKET-789-security-patch
release/v1.2.0
```

## Commit Message Format

```
type(scope): description

[optional body]

[optional footer]
```

### Types
| Type | When to Use |
|------|-------------|
| feat | New feature |
| fix | Bug fix |
| docs | Documentation only |
| style | Formatting, no code change |
| refactor | Code change, no new feature/fix |
| test | Adding tests |
| chore | Build, tooling, maintenance |

### Examples
```
feat(auth): add OAuth2 login support

Implements OAuth2 flow for Google and GitHub.
Adds token refresh and secure storage.

Closes #123
```

```
fix(api): handle null user in response

Prevents 500 error when user is deleted but
session remains active.

Fixes #456
```

## Common Operations

### Feature Branch Workflow
```bash
# Start feature
git checkout -b feature/TICKET-123-description main
git push -u origin feature/TICKET-123-description

# Keep up to date
git fetch origin main
git rebase origin/main

# Finish (after PR approved)
git checkout main
git pull
git merge --no-ff feature/TICKET-123-description
git push
git branch -d feature/TICKET-123-description
```

### Conflict Resolution
```bash
# During rebase
git fetch origin main
git rebase origin/main
# Fix conflicts in files
git add <fixed-files>
git rebase --continue

# If things go wrong
git rebase --abort
```

### Cherry-pick
```bash
git cherry-pick <commit-hash>
git cherry-pick -x <commit-hash>  # Add reference to original
```

### Interactive Rebase
```bash
# Squash last 3 commits
git rebase -i HEAD~3
# Change 'pick' to 'squash' for commits to combine
```

## Release Process

### Semantic Versioning
```
MAJOR.MINOR.PATCH
  │     │     │
  │     │     └── Bug fixes (backward compatible)
  │     └──────── New features (backward compatible)
  └────────────── Breaking changes
```

### Release Steps
```bash
# Create release branch
git checkout -b release/v1.2.0 develop

# Version bump
npm version minor  # or manual edit

# Merge to main and tag
git checkout main
git merge --no-ff release/v1.2.0
git tag -a v1.2.0 -m "Release v1.2.0"

# Merge back to develop
git checkout develop
git merge --no-ff release/v1.2.0

# Push everything
git push origin main develop --tags
```

## Branch Protection (GitHub)

```yaml
main:
  required_reviews: 2
  dismiss_stale_reviews: true
  require_code_owners: true
  require_status_checks:
    - build
    - test
    - lint
  require_linear_history: true
```

## Git Hooks

### Pre-commit
```bash
#!/bin/bash
# .git/hooks/pre-commit

npm run lint || exit 1
npm test || exit 1
```

### Commit-msg
```bash
#!/bin/bash
# Validate format: type(scope): description

commit_regex='^(feat|fix|docs|style|refactor|test|chore)\(.+\): .{10,}'
if ! grep -qE "$commit_regex" "$1"; then
    echo "Invalid commit message format"
    echo "Expected: type(scope): description"
    exit 1
fi
```

## Maintenance

```bash
# Clean up merged branches
git branch --merged main | grep -v main | xargs git branch -d

# Prune remote tracking branches
git remote prune origin

# Find large files in history
git rev-list --objects --all | \
  git cat-file --batch-check='%(objectsize) %(rest)' | \
  sort -rn | head -20
```

## PR Guidelines

1. Keep PRs small and focused
2. Write descriptive titles
3. Link related issues
4. Request appropriate reviewers
5. Respond to feedback promptly

### PR Description Template
```markdown
## Summary
Brief description of changes

## Changes
- Change 1
- Change 2

## Testing
How was this tested?

## Checklist
- [ ] Tests pass
- [ ] Docs updated
- [ ] No breaking changes
```

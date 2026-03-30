# Git Operations Reference

All git commands used by the PR Prep & Review skill. Use these exact commands for consistency and reliability.

---

## State Detection Commands

### Verify git repository
```bash
git rev-parse --is-inside-work-tree
# Returns: true | error (not a repo)
```

### Detect detached HEAD
```bash
git symbolic-ref --short HEAD 2>/dev/null || echo "DETACHED"
# Returns: branch name | "DETACHED"
```

### Get current branch name
```bash
git branch --show-current
# Returns: branch name (empty if detached)
```

### Check working tree status
```bash
git status --porcelain
# Empty output = clean tree
# Non-empty = has changes (staged/unstaged/untracked)
```

### Check for staged changes
```bash
git diff --cached --name-only
```

### Check for unstaged changes
```bash
git diff --name-only
```

### Get full status with context
```bash
git status --short --branch
# Shows: branch tracking info + file status
```

---

## Remote Operations

### Fetch all remote branches (safe, no local changes)
```bash
git fetch origin
```

### Fetch specific remote branch
```bash
git fetch origin TARGET_BRANCH:refs/remotes/origin/TARGET_BRANCH
```

### Verify remote branch exists
```bash
git ls-remote --heads origin TARGET_BRANCH
# Empty output = branch does not exist
# Non-empty = branch exists
```

### List all remote branches
```bash
git branch -r
```

### Check if remote is reachable
```bash
git ls-remote --exit-code origin HEAD 2>&1
# Exit code 0 = reachable; non-zero = network/auth issue
```

---

## Divergence Analysis

### Count commits ahead/behind remote target
```bash
git rev-list --left-right --count origin/TARGET_BRANCH...HEAD
# Output: "BEHIND\tAHEAD" e.g. "3\t5" = 3 behind, 5 ahead
```

### List commits on feature branch not in target
```bash
git log origin/TARGET_BRANCH..HEAD --oneline --no-merges
```

### List commits on target not in feature branch
```bash
git log HEAD..origin/TARGET_BRANCH --oneline --no-merges
```

### Check if branch has been merged into target
```bash
git branch -r --merged origin/TARGET_BRANCH | grep CURRENT_BRANCH
# Output with branch name = already merged
```

---

## Merge Operations

### Dry-run merge (check for conflicts without committing)
```bash
git merge --no-commit --no-ff origin/TARGET_BRANCH
# Check conflicts after: git diff --name-only --diff-filter=U
# Always abort after: git merge --abort
```

### Perform actual merge
```bash
git merge origin/TARGET_BRANCH
```

### Abort a merge in progress
```bash
git merge --abort
```

### Continue a merge after resolving conflicts
```bash
git merge --continue
```

### Merge with specific strategy (prefer ours for conflicts)
```bash
git merge -X ours origin/TARGET_BRANCH   # our version wins
git merge -X theirs origin/TARGET_BRANCH # their version wins
```

---

## Rebase Operations

### Rebase onto remote target
```bash
git rebase origin/TARGET_BRANCH
```

### Abort a rebase in progress
```bash
git rebase --abort
```

### Continue rebase after resolving conflicts
```bash
git rebase --continue
```

### Interactive rebase (last N commits)
```bash
git rebase -i HEAD~N
```

### Check if rebase is in progress
```bash
ls -d .git/rebase-merge .git/rebase-apply 2>/dev/null && echo "REBASE_IN_PROGRESS"
```

---

## Conflict Detection

### List conflicted files (during merge/rebase)
```bash
git diff --name-only --diff-filter=U
# U = Unmerged (conflicted)
```

### Show conflict details in a file
```bash
git diff <filename>
# Shows conflict markers
```

### Check if conflict markers remain in files
```bash
grep -rn "<<<<<<< \|======= \|>>>>>>> " --include="*.ts" --include="*.js" --include="*.py" .
# Adjust extensions as needed
```

### List binary conflicted files
```bash
git diff --name-only --diff-filter=U | while read f; do file "$f"; done | grep -v "text"
```

---

## Stash Operations

### Stash all uncommitted changes
```bash
git stash push -m "pr-prep-auto-stash"
```

### List stashes
```bash
git stash list
```

### Restore most recent stash
```bash
git stash pop
```

### Restore specific stash
```bash
git stash apply stash@{0}
```

### Drop a stash
```bash
git stash drop stash@{0}
```

---

## Diff Analysis

### Summary: changed files and line counts
```bash
git diff origin/TARGET_BRANCH..HEAD --stat
```

### List only changed file names
```bash
git diff origin/TARGET_BRANCH..HEAD --name-only
```

### List changed files with change type
```bash
git diff origin/TARGET_BRANCH..HEAD --name-status
# A = Added, M = Modified, D = Deleted, R = Renamed
```

### Full diff content
```bash
git diff origin/TARGET_BRANCH..HEAD
```

### Diff for specific files
```bash
git diff origin/TARGET_BRANCH..HEAD -- src/auth/session.ts
```

### Diff ignoring whitespace
```bash
git diff origin/TARGET_BRANCH..HEAD -w
```

### Count total lines changed
```bash
git diff origin/TARGET_BRANCH..HEAD --shortstat
# Output: X files changed, Y insertions(+), Z deletions(-)
```

### Check if diff is empty
```bash
git diff origin/TARGET_BRANCH --name-only
# Empty output = no changes
```

---

## Commit Analysis

### Recent commits on feature branch (not in target)
```bash
git log origin/TARGET_BRANCH..HEAD --oneline --no-merges
```

### Detailed commit log with body
```bash
git log origin/TARGET_BRANCH..HEAD --no-merges --format="%H %s%n%b"
```

### Commits with changed files
```bash
git log origin/TARGET_BRANCH..HEAD --no-merges --name-only --format="%s"
```

### Check for poor commit messages (one-word, vague)
```bash
git log origin/TARGET_BRANCH..HEAD --no-merges --format="%s" | \
  grep -iE "^(fix|wip|update|changes|stuff|misc|test|temp|working|done|final|more|cleanup|asdf|xxx)\.?$"
```

---

## Branch Naming

### Validate branch naming convention (example: feature/, fix/, etc.)
```bash
git branch --show-current | grep -E "^(feature|fix|hotfix|chore|refactor|release|bugfix)/.+"
# Non-empty = valid; empty = invalid/unconventional
```

### Extract ticket reference from branch name
```bash
git branch --show-current | grep -oE "[A-Z]+-[0-9]+|#[0-9]+|GH-[0-9]+"
```

---

## PR Detection (requires `gh` CLI)

### Check if gh CLI is available
```bash
which gh 2>/dev/null && echo "available" || echo "unavailable"
```

### List open PRs for current branch
```bash
gh pr list --head CURRENT_BRANCH --base TARGET_BRANCH --state open
```

### View PR details
```bash
gh pr view PR_NUMBER
```

### Get PR diff
```bash
gh pr diff PR_NUMBER
```

### Create a PR
```bash
gh pr create --title "PR_TITLE" --body "PR_BODY" --base TARGET_BRANCH
```

### Create a draft PR
```bash
gh pr create --title "PR_TITLE" --body "PR_BODY" --base TARGET_BRANCH --draft
```

---

## Cleanup Operations

### Restore stash after prep (when stash was auto-applied)
```bash
git stash pop
# If conflict: git stash show -p stash@{0}
```

### Remove local branch after merge
```bash
git branch -d BRANCH_NAME
```

### Prune stale remote tracking branches
```bash
git remote prune origin
```

---

## Diagnostic Commands

### Show last commit details
```bash
git log -1 --format="%H %s %ae %ci"
```

### Show git config for repo
```bash
git config --list --local
```

### Show remote URL
```bash
git remote get-url origin
```

### Check merge strategy config
```bash
git config pull.rebase
git config merge.ff
```

### Show all local branches with last commit
```bash
git branch -v
```

---

## Error Handling Patterns

### Handle fetch failure (network/auth)
```bash
git fetch origin 2>&1
# If exit code != 0: warn "Could not fetch remote. Proceeding with local state."
# If output contains "Authentication failed": warn about credentials
```

### Handle rebase with uncommitted changes (will fail)
```bash
# Always check for clean working tree BEFORE attempting rebase
git status --porcelain
# If output non-empty: stash first, then rebase
```

### Detect if merge/rebase already in progress
```bash
# Check for in-progress operations before starting new ones
ls .git/MERGE_HEAD 2>/dev/null && echo "MERGE_IN_PROGRESS"
ls -d .git/rebase-merge 2>/dev/null && echo "REBASE_IN_PROGRESS"
```

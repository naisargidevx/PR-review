---
name: pr-prep-reviewer
description: PR preparation and PR review automation for developer workflows. Helps developers sync branches, detect merge conflicts, generate structured PR titles and descriptions, update README when needed, generate Mermaid flow diagrams for new features, and review PRs for correctness, security, test coverage, and readability. Trigger when user says "prepare my branch for PR", "sync with base branch", "generate PR description", "review this PR", "check for merge conflicts", "create PR title", "review PR before raise", "check README needs update", or "generate flow diagram for feature".
---

# PR Prep & Review Agent

Automate PR preparation and review. Two distinct modes — detect which one the user needs, then follow the corresponding workflow precisely.

> **Config file:** Before running, check for `.pr-agent.yml` in the repo root. Read `references/git-operations.md` for all git commands. Read `references/pr-templates.md` for PR description formats. Read `references/review-checklist.md` for review criteria. Read `references/diagram-templates.md` for Mermaid templates.

---

## Step 0 — Mode Detection

**Always ask the user upfront if intent is ambiguous.** Otherwise detect from their message:

| User says | Mode |
|-----------|------|
| "prepare branch", "sync with base", "create PR", "generate PR description", "check conflicts" | **Mode 1 — PR Preparation** |
| "review PR", "review this diff", "check before raise", "review my changes", "look at this PR" | **Mode 2 — PR Review** |
| "prepare AND review", "full workflow" | **Mode 1 → then Mode 2** |

**Always ask for the target/base branch if not provided.** Never assume it is `main`.

---

## Mode 1 — PR Preparation Workflow

### Phase 1: Git State Validation

Run these checks in order. Stop and report if any critical check fails.

**1.0 Check git is installed**
```bash
git --version 2>&1
```
If command not found → stop: `Git is not installed. Install it from https://git-scm.com and retry.`
If version is below 2.x → warn: `⚠ Git version is outdated. Some commands may behave unexpectedly. Consider upgrading.`

**1.1 Verify git repo**
```bash
git rev-parse --is-inside-work-tree
```
If not a git repo → stop with: `Not inside a git repository. Navigate to your project root and retry.`

**1.1a Check remote named `origin` exists**
```bash
git remote get-url origin 2>&1
```
- If `origin` not found → list all remotes: `git remote -v`
  - If other remotes exist (e.g. `upstream`, `fork`) → ask user: `No remote named 'origin' found. Available remotes: [list]. Which one should be used as origin?`
  - If no remotes at all → stop: `No remotes configured. Add one with: git remote add origin <url>`

**1.2 Detect detached HEAD**
```bash
git symbolic-ref --short HEAD 2>/dev/null || echo "DETACHED"
```
If DETACHED → stop: `You are in detached HEAD state. Checkout a branch before preparing a PR.`

**1.3 Get current branch**
```bash
git branch --show-current
```
Save as `CURRENT_BRANCH`.

**1.4 Validate current branch ≠ target branch**
If `CURRENT_BRANCH == TARGET_BRANCH` → stop: `You are already on the target branch (TARGET_BRANCH). Switch to your feature branch first.`

**1.5 Check working tree cleanliness**
```bash
git status --porcelain
```
- If uncommitted changes exist → warn user clearly:
  ```
  ⚠ Working tree has uncommitted changes:
  [list files]

  Options:
  (1) Stash changes and continue
  (2) Commit changes first
  (3) Abort
  ```
  Ask which option. If stash: `git stash push -m "pr-prep-auto-stash"` and note to restore after.

**1.6 Verify target branch exists on remote**
```bash
git fetch origin 2>&1
git ls-remote --heads origin TARGET_BRANCH
```
Inspect fetch output before proceeding:
- If output contains `Permission denied` or `Authentication failed` → stop:
  ```
  ⚠ Remote authentication failed. Possible fixes:
    - SSH: run `ssh-add ~/.ssh/id_rsa` (or your key path)
    - HTTPS: run `git credential approve` or re-enter credentials
    - Check remote URL: git remote get-url origin
  ```
- If output contains `Could not resolve host` or `network` errors → warn:
  ```
  ⚠ Could not reach remote. Continuing with local cache — remote state may be stale.
  ```
  Continue with locally cached `origin/TARGET_BRANCH` if it exists; otherwise stop.
- If target branch not found on remote → stop: `Target branch 'TARGET_BRANCH' not found on remote. Check the branch name.`

**1.7 Check if current branch already merged**
```bash
git branch -r --merged origin/TARGET_BRANCH | grep CURRENT_BRANCH
```
If found → warn: `⚠ Branch 'CURRENT_BRANCH' appears to already be merged into TARGET_BRANCH.`

**1.8 Check if PR already exists (if gh CLI available)**

First verify gh CLI is authenticated:
```bash
which gh 2>/dev/null && gh auth status 2>&1
```
- If gh not installed → skip steps 1.8 and 1.9 silently.
- If `gh auth status` fails → warn: `⚠ gh CLI is installed but not authenticated. Run: gh auth login` — then skip PR detection.

If gh is available and authenticated:
```bash
gh pr list --head CURRENT_BRANCH --base TARGET_BRANCH --state open 2>/dev/null
```
If open PR exists → warn with PR link: `⚠ An open PR already exists for CURRENT_BRANCH → TARGET_BRANCH: [URL]`

**1.9 Check for already merged or closed PR**
```bash
gh pr list --head CURRENT_BRANCH --base TARGET_BRANCH --state merged 2>/dev/null
gh pr list --head CURRENT_BRANCH --base TARGET_BRANCH --state closed 2>/dev/null
```
If merged PR found → warn: `⚠ A PR from CURRENT_BRANCH into TARGET_BRANCH was already merged. Verify this is intentional (e.g. re-work or hotfix).`
If closed (not merged) PR found → warn: `⚠ A previously closed (unmerged) PR exists for this branch. You may be re-raising a previously rejected change.`

---

### Phase 2: Sync with Target Branch

**2.1 Load sync strategy from config**
Check `.pr-agent.yml` for `sync_strategy`. Default: `merge`. Options: `merge` | `rebase`.

**2.2 Fetch latest remote state**
```bash
git fetch origin TARGET_BRANCH
```

**2.3 Check divergence**
```bash
git rev-list --left-right --count origin/TARGET_BRANCH...HEAD
```
Output: `BEHIND AHEAD`
- If BEHIND > 50 → warn: `⚠ Branch is 50+ commits behind. This may be a large sync — review carefully.`
- If BEHIND == 0 → skip sync, branch is already up to date.

**2.4a Sync via MERGE (default)**
```bash
git merge --no-commit --no-ff origin/TARGET_BRANCH
```
Check for conflicts:
```bash
git diff --name-only --diff-filter=U
```

**2.4b Sync via REBASE**
```bash
git rebase origin/TARGET_BRANCH
```
Check for conflicts:
```bash
git diff --name-only --diff-filter=U
```

**2.5 CONFLICT DETECTED → STOP IMMEDIATELY**

If any conflicts are found, abort the merge/rebase and display:

```
🚫 MERGE CONFLICT DETECTED — PR PREPARATION HALTED

Conflicts in the following files:
  • src/auth/login.ts         (both modified)
  • config/settings.yaml      (deleted by us, modified by them)

What to do next:
  1. Resolve conflicts manually in each file listed above
  2. Remove all conflict markers (<<<<<<<, =======, >>>>>>>)
  3. Stage resolved files: git add <file>
  4. Complete the sync:
     - For merge:  git merge --continue
     - For rebase: git rebase --continue
  5. Re-run this skill after conflicts are resolved

Aborting merge/rebase now...
```

Then run:
```bash
git merge --abort    # if merge strategy
# OR
git rebase --abort   # if rebase strategy
```

**Do not continue to Phase 3 if conflicts exist.**

**2.6 Check for binary file conflicts**
```bash
git diff --name-only --diff-filter=U | xargs file 2>/dev/null | grep -v "text"
```
If binary conflicts found → call out specifically: `Binary file conflicts require manual resolution: [files]`

**2.7 Restore stash if it was auto-applied (step 1.5)**
```bash
git stash pop
```
If stash pop results in conflicts:
```bash
git diff --name-only --diff-filter=U
```
If conflicts found → warn immediately:
```
⚠ Auto-stash restore caused conflicts. Your stash was NOT automatically applied cleanly.
Conflicted files:
  • [list files]
To resolve:
  1. Fix conflicts in each file listed above
  2. Run: git stash drop stash@{0}   (once you're done)
  Your original uncommitted changes are still in: git stash show stash@{0}
```
Do not continue silently — always surface stash pop failures.

**2.8 Check if diff is empty after sync**
```bash
git diff origin/TARGET_BRANCH --name-only
```
If empty → warn: `⚠ No diff detected between CURRENT_BRANCH and TARGET_BRANCH. The PR would be empty. Verify the correct branches are being compared.`

---

### Phase 3: Change Analysis

Run all analysis commands in parallel where possible.

**3.0 Guard: verify commits exist ahead of target**
```bash
git rev-list --count origin/TARGET_BRANCH..HEAD
```
If count is 0 → warn and stop:
```
⚠ No commits found ahead of TARGET_BRANCH.
Your branch has no new commits to include in a PR.
Make sure you are on the correct branch, or that your commits have been pushed:
  git log --oneline -5
```
Do not proceed to generate PR content for an empty commit range.

**3.1 Get commit log**
```bash
git log origin/TARGET_BRANCH..HEAD --oneline --no-merges
```

**3.2 Get full diff**
```bash
git diff origin/TARGET_BRANCH..HEAD --stat
```

**3.3 Get changed files by type**
```bash
git diff origin/TARGET_BRANCH..HEAD --name-only
```
Categorize files mentally:
- Source code (`.ts`, `.js`, `.py`, `.go`, `.java`, `.rb`, etc.)
- Tests (`*.test.*`, `*.spec.*`, `__tests__/`, `test/`)
- Config (`.yaml`, `.yml`, `.env.example`, `.json` at root)
- Docs (`README*`, `*.md`, `docs/`)
- Schema / migration (`migrations/`, `*.sql`, `schema.*`)
- CI/CD (`.github/`, `.gitlab-ci.yml`, `Jenkinsfile`)
- Dependencies (`package.json`, `go.mod`, `requirements.txt`, `Gemfile`)
- Assets (images, fonts, static files)

**Auto-filter noise files before analysis:**
The following file types must be excluded from diff analysis and review. Note them separately as "excluded" but never analyze their content:
- Lock files: `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Gemfile.lock`, `poetry.lock`, `go.sum`
- Generated/compiled: `dist/`, `build/`, `out/`, `*.min.js`, `*.min.css`, `*.bundle.js`
- Binary files: images (`.png`, `.jpg`, `.gif`, `.ico`, `.svg` with binary content), fonts (`.woff`, `.ttf`, `.eot`)

If lock files are the *only* changed files → classify as `dependency-bump` and note: `[Lock file only — dependency versions updated]`
If generated/binary files dominate (>80% of changed files) → warn: `⚠ Most changed files are generated or binary. Ensure source files were also committed.`

**3.4 Read diff content for key changed files**
```bash
git diff origin/TARGET_BRANCH..HEAD -- <important files>
```
Focus on: files with the most logic changes, test files, config files. Skip: generated files, lockfiles, large assets.

**3.5 Detect change type**
Based on commit messages, branch name, and changed files, classify:

| Evidence | Type |
|----------|------|
| Only test files changed | `test` |
| Only `.md` / doc files | `docs` |
| Dependency file only | `dependency-bump` |
| Files renamed/moved, no logic change | `refactor` |
| New files added with feature logic | `feature` |
| Bug-related keywords in branch/commits | `bugfix` |
| Mix of feature + fix + refactor | `mixed` |
| Formatting/lint only | `chore` |

**3.6 Detect ticket/issue reference**
Check for patterns in branch name and commits:
- `JIRA-1234`, `GH-123`, `#123`, `ISS-456`, `ABC-789`
- Extract if found. Use in PR title.

**3.7 Detect if README should be updated**
Update README only when:
- Change type is `feature`
- New endpoints, CLI commands, config keys, environment variables added
- Developer setup or run instructions changed
- Integration behavior changed

Do NOT update README for: bugfixes, refactors, formatting, internal-only changes.

**3.8 Detect if flow diagram is needed**
Generate Mermaid diagram only when:
- Change type is `feature` or `mixed` with new feature
- New user flow, API interaction, service dependency, or data flow exists

---

### Phase 4: PR Content Generation

Load `references/pr-templates.md` for the full template. Apply the rules below.

**4.1 Generate PR title**

Format: `[TYPE] Short description (Ticket if available)`

Rules:
- Under 72 characters
- Title case
- Lead with change type: `Fix:`, `Add:`, `Update:`, `Refactor:`, `Chore:`, `Bump:`
- Include ticket reference if found: `Fix: Resolve login timeout on session expiry [JIRA-1234]`
- If commit messages are poor (e.g. "fix", "wip", "changes"), derive title from diff analysis — note this is inferred
- For mixed PRs: `Fix + Refactor: Auth flow cleanup and session timeout fix`

**4.2 Generate PR description**

Use the structured template from `references/pr-templates.md`. Adapt sections based on change type:

- `bugfix` → full bug template (Issue, Root Cause, Fix, Why it Works, Impact, Testing)
- `feature` → feature template (What was added, Why, Flow, Impact, Testing, Docs)
- `refactor` → refactor template (What changed, Why, What did NOT change, Risk, Testing)
- `chore/dependency-bump` → minimal template (What bumped, Why, Compatibility)
- `mixed` → combine relevant sections, clearly split per concern

**Accuracy rules:**
- Label inferred context as `[Inferred from diff]`
- Do not invent business context not supported by code, commits, or branch name
- If root cause is genuinely unclear, write: `Root cause: [Could not be determined from diff — author should clarify]`
- Never include secrets, tokens, internal config values, or PII in PR text
- If diff is very large (>500 lines), summarize by file group rather than line-by-line

**4.3 Generate Mermaid diagram (feature/mixed only)**

If flow diagram is warranted, load `references/diagram-templates.md` and generate appropriate diagram:
- Flowchart for process/decision flows
- Sequence diagram for API/service interactions
- Component diagram for architectural changes

If code intent is unclear, prefix with: `> **Note:** Best-effort generated flow — please verify accuracy.`

**4.4 README update (feature only, when needed)**

- Read existing README if present
- Identify the relevant section to update (setup, usage, API, config)
- Append or update ONLY the affected section — do not rewrite full README
- If no README exists → create a minimal one covering: project description, setup, usage, relevant new feature
- For monorepos: check for README at both root and package level; update the most relevant one
- Always show the user the proposed README diff and ask for confirmation before writing

**4.5 Assess confidence and suggest draft PR**

Suggest draft PR when:
- Commit messages are all poor quality (no context)
- Change type is `mixed` with unclear separation
- Diff is very large with many unrelated changes
- Root cause could not be determined
- Tests are missing for changed logic

```
⚠ Suggestion: Raise as a DRAFT PR because:
  - [reason 1]
  - [reason 2]
This allows reviewers to comment before it's marked ready.
```

---

### Phase 5: Output Summary

Present the full PR preparation summary:

```
✅ PR PREPARATION COMPLETE

Branch:          feature/auth-timeout-fix
Target:          develop
Commits:         3 commits ahead, 0 behind
Sync:            Merged with origin/develop (no conflicts)
Change type:     bugfix
Ticket:          JIRA-4521

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PR TITLE:
Fix: Resolve session timeout on expired JWT tokens [JIRA-4521]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PR DESCRIPTION:
[full structured description]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
README UPDATE:  Not needed (internal fix)
FLOW DIAGRAM:   Not applicable
DRAFT PR:       Not recommended

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
NEXT STEPS:
  1. Review the PR description above
  2. Create PR: gh pr create --title "..." --body "..."
  3. Assign reviewers and labels per team convention
```

---

## Mode 2 — PR Review Workflow

Used to review a PR diff before or after it is raised.

### Phase 1: Get the Diff

**Option A — Current branch diff against base**
```bash
git diff origin/TARGET_BRANCH..HEAD
```
Before fetching diff, check 0-commits guard:
```bash
git rev-list --count origin/TARGET_BRANCH..HEAD
```
If count is 0 → stop: `⚠ No commits ahead of TARGET_BRANCH. Nothing to review.`

**Option B — User pastes diff** → analyze pasted content directly

**Option C — PR URL or PR number provided**

First check if gh CLI is available and authenticated:
```bash
which gh 2>/dev/null && gh auth status 2>&1
```
- If gh available and authenticated:
  ```bash
  gh pr diff PR_NUMBER
  gh pr view PR_NUMBER
  ```
- If gh not installed or not authenticated → do not stop. Fall back:
  ```
  ⚠ gh CLI is not available or not authenticated. Cannot fetch PR diff automatically.
  Please paste the diff directly into the chat:
    1. Open the PR on GitHub
    2. Click "Files changed"
    3. Copy the diff content and paste it here
  ```

### Phase 2: Load Review Checklist

Read `references/review-checklist.md` fully before reviewing. It contains all criteria by category.

### Phase 3: Systematic Review

**Pre-review filter — exclude noise files:**
Before analyzing, remove the following from scope and note them as excluded:
- Lock files: `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `go.sum`, etc.
- Compiled/minified: `*.min.js`, `*.min.css`, `dist/`, `build/`, `*.bundle.*`
- Binary files: images, fonts, compiled artifacts

If any of the above are present in the diff, state at the top of the review:
```
📎 Excluded from review (noise/generated): [list files]
```

Analyze the diff against every category in the checklist:

1. **Correctness** — Logic errors, off-by-one, wrong conditions, race conditions
2. **Security** — Secrets in code, SQL injection, XSS, auth bypass, missing input validation, exposed config
3. **Readability** — Unclear naming, magic numbers, deeply nested logic, missing comments on non-obvious code
4. **Maintainability** — Dead code, duplicate logic, over-complex abstractions, missing separation of concerns
5. **Test coverage** — Missing tests for new logic, incomplete test updates, flaky assertions, no edge-case tests
6. **Documentation** — API changes undocumented, README not updated, missing migration notes, config changes unexplained
7. **Backward compatibility** — Breaking API changes, removed fields, changed behavior, missing deprecation notice
8. **Performance** — N+1 queries, missing indexes, synchronous blocking calls, memory leaks
9. **Error handling** — Missing null checks, unhandled promise rejections, uncaught exceptions, silent failures
10. **Dependencies** — Unnecessary new deps, known vulnerable packages, license issues

### Phase 4: Review Output

Structure the review output exactly as follows:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PR REVIEW REPORT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📋 SUMMARY
[2-4 sentence overview of what the PR does, overall quality assessment]

🚦 MERGE READINESS: [READY | NEEDS CHANGES | BLOCKED]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔴 CRITICAL ISSUES  (must fix before merge)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. [File:Line] — [Clear description of the issue]
   Why it matters: [impact]
   Suggested fix: [concrete suggestion]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🟡 MEDIUM PRIORITY  (should fix, not blocking)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. [File:Line] — [Issue description]
   Suggestion: [what to do]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔵 MINOR / SUGGESTIONS  (nice to have)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• [Minor issues, style suggestions, optional improvements]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🧪 TEST COVERAGE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[ ] Tests added for new logic: [YES / NO / PARTIAL]
[ ] Edge cases covered:       [list missing cases if any]
[ ] Existing tests updated:   [YES / NO / N/A]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📄 DOCUMENTATION IMPACT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[ ] README update needed:         [YES / NO]
[ ] API docs update needed:       [YES / NO / N/A]
[ ] Migration notes needed:       [YES / NO / N/A]
[ ] Config/env docs update needed:[YES / NO / N/A]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
❓ QUESTIONS FOR AUTHOR
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. [Specific question about intent, design decision, or unclear logic]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💡 SUGGESTED PR TITLE / DESCRIPTION IMPROVEMENTS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[If PR title/description is weak or missing sections, suggest improvements here]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ VERDICT: [APPROVE | REQUEST CHANGES | NEEDS DISCUSSION]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Configuration — `.pr-agent.yml`

The skill reads `.pr-agent.yml` from the repo root if present. All fields are optional with sensible defaults.

**Config parse error fallback:**
If `.pr-agent.yml` exists but contains invalid YAML or unrecognized keys → warn and continue with defaults:
```
⚠ Could not parse .pr-agent.yml — using built-in defaults.
Error: [describe parse issue]
Fix: validate your YAML at https://yaml-online-parser.appspot.com
```
Never stop execution due to a malformed config file.

```yaml
# .pr-agent.yml — PR Agent Configuration
# Place this file in your repo root

pr:
  # Default base/target branch (still ask user if not specified in command)
  default_base_branch: develop

  # Sync strategy: "merge" or "rebase"
  sync_strategy: merge

  # Suggest draft PR automatically when confidence is low
  draft_on_low_confidence: true

  # Ticket extraction regex patterns (applied to branch name and commit messages)
  ticket_patterns:
    - "[A-Z]+-\\d+"        # JIRA style: ABC-123
    - "#\\d+"              # GitHub issue: #123
    - "GH-\\d+"            # GitHub explicit: GH-123

  # Branch naming validation (optional — warn if not matching)
  branch_convention: "^(feature|fix|chore|refactor|hotfix|release)/.+"

  # PR template sections to always include
  template_sections:
    - issue
    - root_cause
    - fix
    - impact
    - testing

  # Mandatory labels to remind about (informational only)
  required_labels: []

  # Reviewer handle suggestions (informational only)
  default_reviewers: []

readme:
  # When to auto-update README: "feature" | "always" | "never"
  update_policy: feature

  # Target README file (relative to repo root)
  # Use array for monorepos
  readme_paths:
    - README.md

diagram:
  # Preferred diagram type: "flowchart" | "sequence" | "component" | "auto"
  preferred_type: auto

  # Include in PR description automatically
  include_in_pr: true

review:
  # Minimum required sections in review output
  required_sections:
    - critical
    - tests
    - docs
    - verdict
```

---

## Edge Case Quick Reference

| Situation | Action |
|-----------|--------|
| Git not installed | Stop, give install link |
| Git version < 2.x | Warn about potential command failures |
| Not in a git repo | Stop immediately with clear message |
| Detached HEAD | Stop, tell user to checkout a branch |
| Current branch = target branch | Stop, cannot PR to self |
| `origin` remote missing | List available remotes, ask which to use |
| No remotes at all | Stop, instruct to add remote |
| Uncommitted changes | Warn, ask user: stash / commit / abort |
| Stash pop causes conflicts | Warn explicitly, show conflicted files, do not hide |
| Target branch not on remote | Stop, verify branch name |
| Branch already merged | Warn, may be duplicate work |
| PR already open | Warn with existing PR link |
| PR previously merged | Warn: may be re-work or duplicate |
| PR previously closed (unmerged) | Warn: previously rejected change |
| gh CLI not authenticated | Warn, suggest `gh auth login`, skip PR detection |
| gh CLI not available | Skip PR detection, skip PR creation commands |
| Fetch authentication failure (SSH/HTTPS) | Stop with specific fix instructions |
| Fetch network failure | Warn, continue with local cache if available |
| Merge conflict detected | Stop, abort merge/rebase, show conflict details |
| Binary file conflict | Call out explicitly |
| Empty diff after sync | Warn, may be no-op PR |
| 0 commits ahead of target | Stop, warn branch has nothing to PR |
| Branch 50+ commits behind | Warn before syncing |
| Lock files only changed | Classify as dependency-bump automatically |
| Generated/binary files dominate diff | Warn, check source files were committed |
| Poor commit messages | Derive title/desc from diff, label as inferred |
| Mixed-purpose PR | Split description clearly by concern |
| Secret detected in diff | Flag immediately as CRITICAL in review |
| No README exists | Create minimal one for features only |
| Monorepo | Check both root and package-level README |
| `.pr-agent.yml` parse error | Warn, continue with built-in defaults |

---

## Important Behavioral Rules

1. **Never assume target branch is `main`** — always ask if not provided
2. **Never continue past a conflict** — abort and show clear instructions
3. **Never fabricate business context** — only use what is evident from diff, commits, and branch name
4. **Label all inferences** — `[Inferred from diff]`, `[Inferred from branch name]`
5. **Never include secrets in PR text** — scan for tokens/keys in diff and redact
6. **Prefer draft PR when confidence is low** — better than a misleading PR
7. **Keep README edits minimal and scoped** — only the relevant section, always confirm before writing
8. **Prefer Mermaid diagrams** — never generate vague text-only "mind maps"
9. **Review comments must be actionable** — always include "Suggested fix" for critical/medium issues
10. **Adapt template to change type** — do not force bug-fix language onto a refactor or feature PR

---

## References

- `references/pr-templates.md` — Full PR description templates for each change type
- `references/review-checklist.md` — Comprehensive review criteria by category
- `references/git-operations.md` — Git commands for all operations in this skill
- `references/diagram-templates.md` — Mermaid diagram templates for common flow types

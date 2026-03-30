# pr-prep-reviewer

A Claude Code skill for PR preparation and PR review automation. Helps developer teams prepare branches for merge safely and generate high-quality PR content.

## What It Does

**Mode 1 — PR Preparation**
- Validates git state (clean tree, correct branch, no detached HEAD)
- Fetches latest remote changes and syncs feature branch with the target branch
- Detects merge/rebase conflicts early — stops immediately if conflicts exist
- Analyzes commits, changed files, and diff to classify the change type
- Generates a strong PR title and a structured, template-driven PR description
- Detects if README needs updating (features only — not for trivial fixes)
- Generates Mermaid flow diagrams for new feature flows
- Suggests draft PR when confidence is low

**Mode 2 — PR Review**
- Reviews code diff across 10 categories: correctness, security, readability, maintainability, tests, docs, backward compatibility, performance, error handling, dependencies
- Produces a structured review report with Critical / Medium / Minor findings
- Flags secrets, auth regressions, missing tests, undocumented API changes
- Gives a clear merge readiness verdict

## Usage

### Short Commands (Recommended)

Use these slash commands for the fastest workflow. All commands accept an optional branch name — defaults to `main` if omitted.

| Command | What it does | When to use |
|---------|-------------|-------------|
| `/cpr` | Prepare branch for PR → `main` | Before raising a PR — syncs, checks conflicts, generates title + description |
| `/cpr develop` | Prepare branch for PR → `develop` | Same as above but targets a different base branch |
| `/rpr` | Review current branch diff vs `main` | After writing code, before raising — catches issues early |
| `/rpr develop` | Review diff vs `develop` | Review against a non-main base |
| `/fpr` | Full workflow: prep + review → `main` | When you want everything in one shot |
| `/fpr develop` | Full workflow → `develop` | Full workflow against a specific branch |
| `/spr` | Sync with `main` only | When you just want to bring your branch up to date, no PR content |
| `/spr release/1.4` | Sync with any branch | Sync against a specific release or feature branch |

**Decision guide — which command to use:**
```
Just want to sync my branch?         → /spr
Ready to raise a PR?                 → /cpr
Want to review before raising?       → /rpr
Want both at once?                   → /fpr
Targeting a branch other than main?  → append the branch name: /cpr develop
```

### Natural Language Triggers

You can also describe what you need in plain text:

```
"Prepare my branch for PR against develop"
"Sync with release/1.4 before PR"
"Generate PR title and description from my current diff"
"Review this PR before I raise it"
"Check if README needs updating"
"Generate a flow diagram for the new feature"
"Detect merge conflicts before PR creation"
```

## Configuration

Add a `.pr-agent.yml` to your repo root to configure the skill:

```yaml
pr:
  default_base_branch: develop
  sync_strategy: merge       # or: rebase
  draft_on_low_confidence: true
  ticket_patterns:
    - "[A-Z]+-\\d+"
    - "#\\d+"

readme:
  update_policy: feature     # or: always | never

diagram:
  preferred_type: auto       # or: flowchart | sequence | component
```

## Files

```
pr-prep-reviewer/
├── SKILL.md                          # Main skill instructions
├── README.md                         # This file
└── references/
    ├── pr-templates.md               # PR description templates by change type
    ├── review-checklist.md           # Complete review criteria (10 categories)
    ├── git-operations.md             # All git commands used by the skill
    └── diagram-templates.md          # Mermaid diagram templates
```

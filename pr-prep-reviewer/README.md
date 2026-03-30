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

Trigger the skill by describing what you need:

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

---
name: ci-review-gate
description: GitHub Actions CI/CD integration that auto-triggers code review on every push to a feature branch, applies fixes automatically, and then runs the CI pipeline. Ensures no code reaches CI without passing review first.
metadata:
  author: Ashish
  version: "1.0"
  type: infrastructure
  persona: CI Gate
  requires: github-actions
---

# CI Review Gate Skill

An **infrastructure skill** that integrates the code review pipeline into GitHub Actions CI/CD. Every push to a feature branch automatically triggers a review, applies fixable issues, and only then runs the CI pipeline.

## Flow

```
Developer pushes to feature branch
         │
         ▼
┌─────────────────────────────┐
│  Phase 1: Review & Auto-fix │
│                             │
│  1. Get diff (base → HEAD)  │
│  2. Run orchestrator triage │
│  3. Run specialist reviews  │
│  4. Apply fixable findings  │
│  5. Commit & push fixes     │
│  6. Post review to PR       │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│  Phase 2: CI/CD Pipeline    │
│                             │
│  1. Checkout (with fixes)   │
│  2. Install dependencies    │
│  3. Lint                    │
│  4. Test                    │
│  5. Build                   │
└─────────────────────────────┘
```

## How It Works

### Trigger

The workflow triggers on:
- **Push** to any branch except `main`, `master`, and `release/**`
- **Pull request** opened, updated, or reopened

```yaml
on:
  push:
    branches-ignore:
      - main
      - master
      - release/**
  pull_request:
    types: [opened, synchronize, reopened]
```

### Phase 1: Review & Auto-fix

1. **Checkout** the code with full git history (`fetch-depth: 0`)
2. **Get the diff** between the base commit and HEAD
3. **Run Claude Code** with the orchestrator skill to triage and review
4. **Parse findings** — extract fixable issues (those with `fix_code` and `original_code`)
5. **Apply fixes** — replace original code with fix code in each file
6. **Commit & push** — commit all fixes as a single commit back to the branch
7. **Post review** — on PRs, post a review comment with findings summary

### Phase 2: CI/CD Pipeline

Runs **after** Phase 1 completes. Checks out the latest code (including any auto-fix commits) and runs the standard CI steps.

## Setup

### Required Secrets

| Secret | Required | Purpose |
|--------|----------|---------|
| `ANTHROPIC_API_KEY` | Yes | API key for Claude Code to run reviews |
| `GITHUB_TOKEN` | Auto | Provided by GitHub Actions — used for PR comments and pushing fixes |

### Required Permissions

The workflow needs these permissions in the repository:

```yaml
permissions:
  contents: write        # Push auto-fix commits
  pull-requests: write   # Post PR reviews
  issues: write          # Comment on issues
```

### Installation

1. Copy `.github/workflows/code-review.yml` to your repository
2. Add `ANTHROPIC_API_KEY` to your repository secrets (Settings → Secrets → Actions)
3. Customize the CI/CD steps in Phase 2 for your project (lint, test, build)

## Review Output Schema

The review step outputs findings as JSON:

```json
{
  "passed": false,
  "summary": "Found 2 security issues and 1 performance concern.",
  "findings": [
    {
      "severity": "blocker",
      "category": "security",
      "file": "src/auth/login.ts",
      "line": 42,
      "description": "SQL injection via string concatenation",
      "original_code": "db.query(`SELECT * FROM users WHERE id = ${userId}`)",
      "fix_code": "db.query('SELECT * FROM users WHERE id = ?', [userId])"
    },
    {
      "severity": "major",
      "category": "security",
      "file": "src/api/upload.ts",
      "line": 15,
      "description": "Path traversal — user controls file path",
      "original_code": "fs.readFile(baseDir + req.params.file)",
      "fix_code": "fs.readFile(path.join(baseDir, path.basename(req.params.file)))"
    },
    {
      "severity": "minor",
      "category": "performance",
      "file": "src/services/report.ts",
      "line": 88,
      "description": "N+1 query inside loop",
      "original_code": null,
      "fix_code": null
    }
  ]
}
```

### Fix Decision Logic

| Finding | `fix_code` | Action |
|---------|-----------|--------|
| Has exact replacement code | Non-null | Auto-applied and committed |
| Too complex or contextual | Null | Reported in PR review, manual fix required |
| Architecture concern | Null | Reported in PR review only |

## Auto-fix Commit Format

Fixes are committed as a single commit:

```
fix: auto-fix code review findings

Applied automated fixes for issues found during code review.
Categories: security, performance

Co-Authored-By: codereview-bot <codereview-bot@users.noreply.github.com>
```

## PR Review Comment

When running on a pull request, the workflow posts a review:

```markdown
## Automated Code Review

Found 3 issues: 1 blocker, 1 major, 1 minor.

### Auto-fixes Applied
The following issues were automatically fixed and committed:

- **security**: SQL injection via string concatenation (`src/auth/login.ts:42`)
- **security**: Path traversal — user controls file path (`src/api/upload.ts:15`)

### Remaining Issues (Manual Fix Required)

| Severity | Category | File | Description |
|----------|----------|------|-------------|
| minor | performance | `src/services/report.ts:88` | N+1 query inside loop |
```

### Review Event

| Findings | Event | Meaning |
|----------|-------|---------|
| No blockers or majors (after fixes) | `APPROVE` | Good to merge |
| Has blockers or majors remaining | `REQUEST_CHANGES` | Manual fixes needed |

## Concurrency

The workflow uses concurrency groups to avoid duplicate reviews on rapid pushes:

```yaml
concurrency:
  group: code-review-${{ github.ref }}
  cancel-in-progress: true
```

If a developer pushes twice quickly, the first review is cancelled and only the latest push is reviewed.

## Customization

### Skip Review for Certain Paths

Add path filters to skip review for docs-only or config-only changes:

```yaml
on:
  push:
    branches-ignore:
      - main
    paths-ignore:
      - '**.md'
      - 'docs/**'
      - '.github/**'
```

### Control Which Specialists Run

Set environment variables to control the review scope:

```yaml
env:
  REVIEW_SKIP_STYLE: "true"
  REVIEW_SKIP_TESTING: "true"
  REVIEW_SEVERITY_THRESHOLD: "major"  # Only report major+blocker
```

### Disable Auto-fix

To run review-only without auto-fixing:

```yaml
env:
  REVIEW_AUTO_FIX: "false"
```

### Add Custom CI Steps

Edit Phase 2 in the workflow file. The placeholder steps show where to add your project-specific commands:

```yaml
- name: Install dependencies
  run: npm ci

- name: Lint
  run: npm run lint

- name: Test
  run: npm test

- name: Build
  run: npm run build
```

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Review step times out | Large diff, too many files | Set `REVIEW_SEVERITY_THRESHOLD: "major"` to reduce scope |
| Auto-fix commit creates loop | Push triggers another review | Concurrency group cancels the redundant run |
| Fixes don't apply | `original_code` doesn't match file content | Code may have changed between review and fix; re-push to retry |
| No review posted | Not a PR event | Review comments only post on `pull_request` events |
| Permission denied on push | Token lacks write permission | Ensure `contents: write` permission is set |
| `ANTHROPIC_API_KEY` not found | Secret not configured | Add it in Settings → Secrets and variables → Actions |

## Quick Reference

```
□ Setup
  □ Copy workflow to .github/workflows/
  □ Add ANTHROPIC_API_KEY secret
  □ Customize CI steps in Phase 2

□ Workflow Runs
  □ Push to feature branch → triggers review
  □ Open/update PR → triggers review
  □ Fixable issues → auto-committed
  □ Non-fixable issues → reported in PR review
  □ CI pipeline runs after review completes

□ Review Results
  □ No issues → APPROVE
  □ All issues auto-fixed → APPROVE
  □ Remaining blockers/majors → REQUEST_CHANGES
```

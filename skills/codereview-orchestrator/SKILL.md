---
name: codereview-orchestrator
description: Triage and orchestrate code reviews. Analyzes PR intent, identifies touched surfaces, assesses risk, and routes to specialist skills. Does NOT perform detailed review - delegates to specialists. Supports full pipeline with "Review PR <number>" command.
metadata:
  author: Ashish
  version: "3.0"
  persona: Review Coordinator
---

# Code Review Orchestrator Skill

The coordinator for code reviews. This skill **only** triages and routes - it does NOT perform detailed code review. All actual review work is delegated to specialist skills.

## Quick Start: Full Pipeline

Trigger a complete review by saying:

```
Review PR 123
Review PR owner/repo#123
Review PR https://github.com/owner/repo/pull/123
```

This will:
1. **Retrieve** the PR diff via GitHub API
2. **Triage** and assess risk
3. **Route** to appropriate specialist skills
4. **Review** the code
5. **Submit** the review to GitHub

## Role

- **Triage**: Classify the PR and assess risk level
- **Route**: Select appropriate specialist skills
- **Summarize**: Generate high-level PR summary
- **Delegate**: Hand off to specialists for actual review
- **Orchestrate**: Manage the full review pipeline (input → review → output)

## What This Skill Does NOT Do

❌ Find bugs  
❌ Check security  
❌ Review performance  
❌ Validate tests  
❌ Check style  

All of the above are delegated to specialist skills.

## Full Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         INPUT SKILLS                             │
├─────────────────────────────────────────────────────────────────┤
│  retrieve-diff-from-github-pr  │  retrieve-diff-from-commit     │
│  (GitHub PRs via API)          │  (Local git commits)           │
└────────────────────────────────┴────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                    codereview-orchestrator                       │
│                   (Triage & Route - this skill)                  │
└─────────────────────────────────────────────────────────────────┘
                                 │
     ┌───────┬───────┬───────────┴───────────┬───────┬───────┐
     ▼       ▼       ▼                       ▼       ▼       ▼
┌─────────┐ ┌─────┐ ┌─────┐             ┌─────────┐ ┌─────┐ ┌─────┐
│security │ │ api │ │data │    ...      │  perf   │ │test │ │style│
└─────────┘ └─────┘ └─────┘             └─────────┘ └─────┘ └─────┘
     │       │       │                       │       │       │
     └───────┴───────┴───────────┬───────────┴───────┴───────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                        OUTPUT SKILLS                             │
├─────────────────────────────────────────────────────────────────┤
│                    submit-github-review                          │
│                 (Post review to GitHub API)                      │
└─────────────────────────────────────────────────────────────────┘
```

## Inputs

| Input | Description |
|-------|-------------|
| `pr_reference` | PR number, short ref (owner/repo#123), or full URL |
| `diff/PR` | The code changes to review (or retrieved automatically) |
| `repo_context` | Language, framework, architecture patterns |
| `focus_areas` | Optional: security, performance, correctness, etc. |
| `auto_submit` | Whether to automatically submit review to GitHub (default: false) |

## Outputs

| Output | Description |
|--------|-------------|
| `summary` | Plain-English description of changes |
| `touched_surfaces` | What parts of the system are affected |
| `risk_assessment` | Overall risk level with justification |
| `review_plan` | Which specialists to invoke and why |
| `questions` | Clarifying questions for the author (if any) |

## Step 1: Understand Intent

Ask these questions (do NOT review the code):

- What behavior change is intended?
- Does the PR description explain the purpose?
- Is this a feature, bugfix, refactor, or infrastructure change?

## Step 2: Identify Touched Surfaces

Categorize modified files into surfaces:

| Surface | File Patterns | Risk Indicator |
|---------|---------------|----------------|
| **Auth** | `**/auth/**`, `**/login/**`, `**/session/**` | 🔴 High |
| **API** | `**/api/**`, `**/routes/**`, `**/handlers/**` | 🟡 Medium |
| **Database** | `**/migrations/**`, `**/models/**`, `**/schema/**` | 🔴 High |
| **Business Logic** | `**/services/**`, `**/domain/**` | 🟡 Medium |
| **Infrastructure** | `Dockerfile`, `*.yaml`, `terraform/**` | 🟡 Medium |
| **Configuration** | `**/config/**`, `.env*`, `*.json` | 🟡 Medium |
| **Tests** | `**/test/**`, `**/spec/**`, `**/*.test.*` | 🟢 Low |
| **Documentation** | `*.md`, `**/docs/**` | 🟢 Low |
| **Dependencies** | `package.json`, `requirements.txt`, `go.mod` | 🟡 Medium |

## Step 3: Assess Risk

Rate overall risk based on:

| Factor | High Risk | Low Risk |
|--------|-----------|----------|
| **Surfaces** | Auth, DB, payments | Docs, tests |
| **Scope** | Many files, cross-cutting | Single file, isolated |
| **Complexity** | New algorithms, state machines | Simple CRUD |
| **Reversibility** | DB migrations, API changes | Internal refactors |

## Step 4: Generate Review Plan

Select specialists based on touched surfaces:

```yaml
review_plan:
  # Always run
  always:
    - codereview-correctness   # Logic bugs
    - codereview-style         # Readability
  
  # Conditional based on surfaces
  conditional:
    - skill: codereview-security
      trigger: auth, input handling, secrets, external APIs
      
    - skill: codereview-api
      trigger: routes, endpoints, schemas, contracts
      
    - skill: codereview-data
      trigger: migrations, models, queries
      
    - skill: codereview-concurrency
      trigger: async, workers, queues, locks
      
    - skill: codereview-performance
      trigger: loops, queries, caching, I/O
      
    - skill: codereview-observability
      trigger: logging, metrics, tracing
      
    - skill: codereview-testing
      trigger: test files modified or missing
      
    - skill: codereview-config
      trigger: config files, env vars, feature flags
      
    - skill: codereview-architect
      trigger: core utilities, shared libraries, breaking changes
```

## Output Format

```markdown
## PR Summary

[2-3 sentence description of what this PR does]

## Touched Surfaces

| Surface | Files | Risk |
|---------|-------|------|
| Auth | `auth/login.ts`, `auth/session.ts` | 🔴 High |
| API | `routes/users.ts` | 🟡 Medium |
| Tests | `tests/user.test.ts` | 🟢 Low |

## Risk Assessment

**Overall Risk: 🟡 MEDIUM**

- 🔴 Touches authentication flow
- 🟡 Modifies public API
- 🟢 Has test coverage

## Review Plan

| Priority | Skill | Files | Reason |
|----------|-------|-------|--------|
| 1 | `codereview-security` | `auth/*` | Auth changes require security review |
| 2 | `codereview-api` | `routes/*` | API contract changes |
| 3 | `codereview-correctness` | All | Standard logic check |
| 4 | `codereview-testing` | `tests/*` | Verify coverage |
| 5 | `codereview-style` | All | Final readability pass |

## Questions for Author

1. [Only if something is genuinely unclear about intent]
```

## Specialist Skills Reference

| Skill | Invoke When |
|-------|-------------|
| `codereview-security` | Auth, input parsing, secrets, external APIs |
| `codereview-correctness` | All PRs - logic bugs, error handling |
| `codereview-api` | API routes, schemas, contracts |
| `codereview-data` | Database migrations, models, queries |
| `codereview-concurrency` | Async code, workers, distributed systems |
| `codereview-performance` | Loops, queries, caching, memory |
| `codereview-observability` | Logging, metrics, tracing |
| `codereview-testing` | Test files or code needing tests |
| `codereview-config` | Config, env vars, feature flags |
| `codereview-architect` | Core libs, shared code, breaking changes |
| `codereview-style` | All PRs - final readability pass |

## Quick Reference

```
□ Understand Intent
  □ What does this PR do?
  □ Feature / bugfix / refactor / infra?

□ Identify Surfaces
  □ Which areas are touched?
  □ What's the risk level of each?

□ Assess Risk
  □ Overall risk rating?
  □ Key risk factors?

□ Generate Plan
  □ Which specialists needed?
  □ In what priority order?
  □ Why each specialist?
```

## Important

This skill is **only** for triage and routing. Once the review plan is generated, invoke the specialist skills to perform the actual review.

---

## Full Pipeline Execution

When triggered with "Review PR <number>", execute the full pipeline:

### Phase 1: Input (Retrieve Diff)

```yaml
# For GitHub PRs
skill: retrieve-diff-from-github-pr
inputs:
  owner: <from PR reference>
  repo: <from PR reference>
  pull_number: <from PR reference>
outputs:
  - pr_info
  - files
  - diff
  - commit_id  # Needed for submit phase
```

### Phase 2: Triage (This Skill)

Execute Steps 1-4 above to generate the review plan.

### Phase 3: Review (Specialist Skills)

Execute each specialist skill in the review plan:

```yaml
for each skill in review_plan:
  invoke: <skill>
  inputs:
    diff: <from phase 1>
    files: <relevant files for this skill>
  collect: findings[]
```

### Phase 4: Output (Submit Review)

```yaml
skill: submit-github-review
inputs:
  owner: <from phase 1>
  repo: <from phase 1>
  pull_number: <from phase 1>
  commit_id: <from phase 1>
  findings: <aggregated from phase 3>
  review_event: <determined by findings severity>
outputs:
  - review_url
```

### Pipeline Output

```markdown
## Review Complete

**PR**: owner/repo#123
**Review URL**: https://github.com/owner/repo/pull/123#pullrequestreview-12345

### Summary

| Severity | Count |
|----------|-------|
| 🔴 Blocker | 1 |
| 🟡 Major | 2 |
| 🔵 Minor | 3 |
| ⚪ Nit | 2 |

**Action**: REQUEST_CHANGES

View the full review on GitHub: [PR #123](https://github.com/owner/repo/pull/123)
```

## Input/Output Skills Reference

| Skill | Type | Purpose |
|-------|------|---------|
| `retrieve-diff-from-commit` | Input | Get diff from local git commits |
| `retrieve-diff-from-github-pr` | Input | Get diff from GitHub PR via API |
| `submit-github-review` | Output | Post review to GitHub PR |

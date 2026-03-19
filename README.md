Agent Skills for Code Review.

These skills follow the [Agent Skills specification](https://agentskills.io/specification) so they can be used by any skills-compatible agent, including Claude Code and Codex CLI.

## Quick Start

Review a GitHub PR with a single command:

```
Review PR 123
Review PR owner/repo#123  
Review PR https://github.com/owner/repo/pull/123
```

This will automatically:
1. Fetch the PR diff from GitHub
2. Triage and assess risk
3. Run appropriate specialist reviews
4. Post the review to GitHub

### CI/CD Automation

Add the GitHub Actions workflow to auto-review every push:

```bash
# Copy the workflow to your repo
cp .github/workflows/code-review.yml <your-repo>/.github/workflows/

# Add your Anthropic API key as a repository secret
# Settings → Secrets → Actions → ANTHROPIC_API_KEY
```

Now every push to a feature branch will: **review → auto-fix → then run CI**.

## Overview

This plugin provides a comprehensive code review system modeled after industry-leading tools like CodeRabbit, Cursor BugBot, and Greptile. It uses an **Input → Orchestrator → Specialists → Output** pipeline architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                    CI/CD TRIGGER (optional)                   │
│      Push to feature branch → GitHub Actions workflow        │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                       INPUT SKILLS                           │
│   retrieve-diff-from-github-pr  │  retrieve-diff-from-commit │
└────────────────────────────────┬────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────┐
│                  codereview-orchestrator                     │
│                (Triage & Route - nothing else)               │
└──────────────────────────┬──────────────────────────────────┘
                           │
     ┌───────┬───────┬─────┴─────┬───────┬───────┬───────┐
     ▼       ▼       ▼           ▼       ▼       ▼       ▼
┌─────────┐ ┌─────┐ ┌─────┐ ┌─────────┐ ┌─────┐ ┌─────┐ ┌─────┐
│security │ │ api │ │data │ │concurr. │ │perf │ │test │ │style│
└─────────┘ └─────┘ └─────┘ └─────────┘ └─────┘ └─────┘ └─────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                       OUTPUT SKILLS                          │
│          auto-fix-patches  │  submit-github-review           │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    INTERACTION SKILLS                         │
│                     interactive-chat                          │
│            (Reply to comments, handle pushback)              │
└─────────────────────────────────────────────────────────────┘
```

## Installation

### Marketplace

> **Coming soon** — Marketplace publishing is not yet available. Use manual installation for now.

<!-- ```
/plugin marketplace add ashishsarkar/claude-code-review
/plugin install codereview@claude-code-review
``` -->

### Manually

#### Claude Code

Add the contents of this repo to a `/.claude` folder in your project root (or whichever folder you're using with Claude Code). See more in the [official Claude Skills documentation](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview).

#### Codex CLI

Copy the `skills/` directory into your Codex skills path (typically `~/.codex/skills`). See the [Agent Skills specification](https://agentskills.io/specification) for the standard skill format.

## Skills

### 📥 Input Skills

| Skill | Description | Use When |
|-------|-------------|----------|
| **retrieve-diff-from-github-pr** | Fetch PR diff and metadata via GitHub API | Reviewing GitHub PRs |
| **retrieve-diff-from-commit** | Get diff from local git commits | Reviewing local changes |

### 🎯 Orchestrator

| Skill | Description | Use When |
|-------|-------------|----------|
| **codereview-orchestrator** | Triage, assess risk, route to specialists. Coordinates full pipeline. | Entry point for all reviews |

### 🔍 Specialists

| Skill | Modeled After | Focus Area | Trigger |
|-------|---------------|------------|---------|
| **codereview-security** | Cursor BugBot | Vulnerabilities, auth, injection, secrets | Auth, input, API calls |
| **codereview-correctness** | - | Logic bugs, error handling, edge cases | Core business logic |
| **codereview-api** | - | Contracts, breaking changes, versioning | Routes, endpoints, schemas |
| **codereview-data** | - | Migrations, queries, transactions | Database, models |
| **codereview-concurrency** | - | Retries, idempotency, distributed systems | Async, workers, queues |
| **codereview-performance** | - | O(n²), N+1, memory leaks, caching | Loops, queries, I/O |
| **codereview-observability** | - | Logging, metrics, tracing, alerting | Monitoring code |
| **codereview-testing** | - | Coverage, quality, determinism | Test files |
| **codereview-style** | - | Readability, maintainability, docs | All files (final pass) |
| **codereview-config** | - | Secrets, feature flags, environment | Config, env files |
| **codereview-architect** | Greptile | Blast radius, dependencies, patterns | Core utilities, shared libs |

### 📤 Output Skills

| Skill | Description | Use When |
|-------|-------------|----------|
| **submit-github-review** | Post review findings to GitHub PR via API | Submitting review to GitHub |
| **auto-fix-patches** | Generate one-click GitHub suggestion blocks from findings | Providing actionable code fixes |

### 💬 Interaction Skills

| Skill | Description | Use When |
|-------|-------------|----------|
| **interactive-chat** | Reply to PR comment threads, handle pushback, answer questions, retract false positives | Author replies to a review comment |

### 🔄 CI/CD Integration

| Skill | Description | Use When |
|-------|-------------|----------|
| **ci-review-gate** | GitHub Actions workflow — auto-reviews on push, applies fixes, then runs CI | Automating reviews in CI/CD |

### 📋 Methodology

| Skill | Description |
|-------|-------------|
| **general-codereview** | Google's classic 5-step methodology: Pre-screen → Understand → Verify → Optimize → Check |

## Recommended Workflow

### One-Command Pipeline (GitHub PRs)
```
Review PR 123
```
This single command runs the full pipeline: fetch → triage → review → submit.

### Quick Review (Local)
```
1. retrieve-diff-from-commit  → Get local diff
2. codereview-orchestrator    → Triage & route
3. Run recommended specialists in order
```

### Comprehensive Review
```
1. retrieve-diff-from-github-pr  → Fetch PR diff
2. codereview-orchestrator       → Triage and generate plan
3. codereview-security           → Security issues
4. codereview-correctness        → Logic bugs
5. codereview-api                → Contract changes
6. codereview-data               → Database safety
7. codereview-concurrency        → Distributed concerns
8. codereview-performance        → Optimization
9. codereview-testing            → Test coverage
10. codereview-style             → Final cleanup
11. auto-fix-patches             → Generate one-click fix suggestions
12. submit-github-review         → Post review to GitHub
13. interactive-chat             → Reply to author comments
```

## Finding Schema

All specialist skills output findings in a consistent format:

```json
{
  "severity": "blocker|major|minor|nit",
  "category": "security|correctness|performance|...",
  "evidence": {
    "file": "path/to/file.ts",
    "line": 42,
    "snippet": "problematic code"
  },
  "impact": "What breaks or what's the risk",
  "fix": "Suggested change",
  "test": "What test would catch this"
}
```

## Feature Matrix

### Pipeline Skills

| Feature | GitHub PR Input | Commit Input | GitHub Submit |
|---------|:---------------:|:------------:|:-------------:|
| Fetch PR via API | ✅ | | |
| Get Local Diff | | ✅ | |
| Post Review | | | ✅ |
| Inline Comments | | | ✅ |
| Approve/Request Changes | | | ✅ |
| One-click Fix Suggestions | | | ✅ |

### Review Skills

| Feature | Orchestrator | Security | Correct | API | Data | Concur | Perf | Observe | Test | Style | Config |
|---------|:------------:|:--------:|:-------:|:---:|:----:|:------:|:----:|:-------:|:----:|:-----:|:------:|
| PR Summary | ✅ | | | | | | | | | | |
| Risk Assessment | ✅ | | | | | | | | | | |
| Specialist Routing | ✅ | | | | | | | | | | |
| SQL Injection | | ✅ | | | | | | | | | |
| XSS/SSRF | | ✅ | | | | | | | | | |
| Auth Bypass | | ✅ | | | | | | | | | |
| Secret Detection | | ✅ | | | | | | | | | |
| Logic Bugs | | | ✅ | | | | | | | | |
| Error Handling | | | ✅ | | | | | | | | |
| Edge Cases | | | ✅ | | | | | | | | |
| Breaking Changes | | | | ✅ | | | | | | | |
| API Versioning | | | | ✅ | | | | | | | |
| Migration Safety | | | | | ✅ | | | | | | |
| Query Performance | | | | | ✅ | | | | | | |
| Transaction Safety | | | | | ✅ | | | | | | |
| Retry Logic | | | | | | ✅ | | | | | |
| Idempotency | | | | | | ✅ | | | | | |
| Race Conditions | | | | | | ✅ | | | | | |
| N+1 Detection | | | | | | | ✅ | | | | |
| Memory Leaks | | | | | | | ✅ | | | | |
| Caching | | | | | | | ✅ | | | | |
| Logging Quality | | | | | | | | ✅ | | | |
| Metrics Coverage | | | | | | | | ✅ | | | |
| Tracing | | | | | | | | ✅ | | | |
| Test Coverage | | | | | | | | | ✅ | | |
| Flaky Tests | | | | | | | | | ✅ | | |
| Code Readability | | | | | | | | | | ✅ | |
| Documentation | | | | | | | | | | ✅ | |
| Secret Management | | | | | | | | | | | ✅ |
| Feature Flags | | | | | | | | | | | ✅ |

## Comparison with Industry Tools

| Feature | This Plugin | CodeRabbit | BugBot | Greptile |
|---------|:-----------:|:----------:|:------:|:--------:|
| PR Summary | ✅ | ✅ | | |
| Interactive Chat | ✅ | ✅ | | |
| Security Analysis | ✅ | ✅ | ✅ | |
| Deep Context | ✅ | | | ✅ |
| Auto-fix Patches | ✅ | ✅ | | |
| Low False Positives | ✅ | | ✅ | |
| Self-hosted | ✅ | | | |
| Customizable | ✅ | | | |
| GitHub API Integration | ✅ | ✅ | ✅ | ✅ |
| One-command Pipeline | ✅ | ✅ | | |
| Local Commit Review | ✅ | | | |
| CI/CD Auto-review | ✅ | ✅ | ✅ | |
| Auto-fix & Commit | ✅ | | | |

## License

MIT

## Author

Ashish

---

*Based on "5 steps to perform code reviews at Google" (2013) and modern code review tools like CodeRabbit, Cursor BugBot, and Greptile.*

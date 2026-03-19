---
name: submit-github-review
description: Submit a code review to GitHub via the GitHub API. Use this as the final step in a code review pipeline to post review findings to a PR.
metadata:
  author: Ashish
  version: "1.0"
  type: output
  requires: github-api
---

# Submit GitHub Review Skill

An **output skill** that submits code review findings to GitHub via the API. This is the final step in the review pipeline, posting the review to the PR.

## Role

- **Format**: Transform review findings into GitHub review format
- **Submit**: Post the review via GitHub API
- **Annotate**: Add inline comments to specific lines

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| `owner` | Yes | Repository owner (username or organization) |
| `repo` | Yes | Repository name |
| `pull_number` | Yes | Pull Request number |
| `commit_id` | Yes | SHA of the commit to review (from retrieve-diff-from-github-pr) |
| `findings` | Yes | Array of review findings from specialist skills |
| `review_event` | Optional | APPROVE, REQUEST_CHANGES, or COMMENT (default: COMMENT) |

## Outputs

| Output | Description |
|--------|-------------|
| `review_id` | ID of the created review |
| `review_url` | URL to view the review |
| `comments_posted` | Number of inline comments posted |

## Required MCP Tools

This skill uses the GitHub MCP server with:

| Tool | Purpose |
|------|---------|
| `create_pull_request_review` | Submit the review with body and inline comments |

## Step 1: Aggregate Findings

Collect all findings from specialist skills:

```json
{
  "findings": [
    {
      "severity": "blocker",
      "category": "security",
      "evidence": {
        "file": "src/auth/login.ts",
        "line": 42,
        "snippet": "password = req.body.password"
      },
      "impact": "Password logged in plaintext",
      "fix": "Remove logging or hash before logging",
      "test": "Check logs for sensitive data"
    }
  ]
}
```

## Step 2: Determine Review Event

Based on findings severity, determine the review action:

| Findings | Event | Rationale |
|----------|-------|-----------|
| Any **blocker** | `REQUEST_CHANGES` | PR should not be merged |
| Any **major** | `REQUEST_CHANGES` | Significant issues need fixing |
| Only **minor/nit** | `COMMENT` | Suggestions, not blocking |
| No issues | `APPROVE` | PR looks good |

## Step 3: Format Review Body

Create the review summary:

```markdown
## Code Review Summary

### 🔴 Blockers (X)
| File | Line | Issue |
|------|------|-------|
| src/auth/login.ts | 42 | SQL injection vulnerability |

### 🟡 Major (X)
| File | Line | Issue |
|------|------|-------|
| src/api/users.ts | 15 | Missing error handling |

### 🔵 Minor (X)
- Consider adding JSDoc to public functions
- Unused import on line 3

### 📋 Nits (X)
- Formatting: extra blank line at EOF

---

*Reviewed by codereview-skills*
```

## Step 4: Format Inline Comments

Convert findings to GitHub inline comments:

```json
{
  "comments": [
    {
      "path": "src/auth/login.ts",
      "line": 42,
      "body": "🔴 **Security**: SQL injection vulnerability\n\n```suggestion\nconst user = await db.query('SELECT * FROM users WHERE id = ?', [userId]);\n```\n\n**Impact**: Attacker can execute arbitrary SQL\n**Fix**: Use parameterized queries"
    }
  ]
}
```

### Comment Format

```markdown
<severity_emoji> **<category>**: <title>

<description>

```suggestion
<suggested fix if applicable>
```

**Impact**: <what breaks or the risk>
**Fix**: <how to fix it>
```

Severity emojis:
- 🔴 Blocker
- 🟡 Major
- 🔵 Minor
- ⚪ Nit

## Step 5: Submit Review

Use the GitHub MCP tool:

```json
{
  "tool": "create_pull_request_review",
  "server": "user-github",
  "arguments": {
    "owner": "<owner>",
    "repo": "<repo>",
    "pull_number": <number>,
    "commit_id": "<sha>",
    "body": "<review summary>",
    "event": "REQUEST_CHANGES",
    "comments": [
      {
        "path": "src/auth/login.ts",
        "line": 42,
        "body": "🔴 **Security**: SQL injection..."
      }
    ]
  }
}
```

## Output Format

```json
{
  "status": "success",
  "review": {
    "id": 12345,
    "url": "https://github.com/owner/repo/pull/123#pullrequestreview-12345",
    "event": "REQUEST_CHANGES",
    "body": "## Code Review Summary...",
    "comments_count": 5
  },
  "summary": {
    "blockers": 1,
    "major": 2,
    "minor": 3,
    "nits": 2,
    "total": 8
  }
}
```

## Full Pipeline Integration

This skill is the final step in the review pipeline:

```
1. retrieve-diff-from-github-pr
   ↓ (PR info + diff + commit_id)
2. codereview-orchestrator
   ↓ (triage + routing plan)
3. Specialist skills (parallel or sequential)
   ↓ (findings array)
4. submit-github-review (this skill)
   ↓ (posted review)
5. Return URL to user
```

## Quick Reference

```
□ Aggregate Findings
  □ Collect from all specialist skills
  □ Deduplicate if needed

□ Determine Event
  □ Any blockers/major → REQUEST_CHANGES
  □ Only minor/nit → COMMENT
  □ No issues → APPROVE

□ Format Body
  □ Summary with severity breakdown
  □ Table of issues by severity

□ Format Comments
  □ Convert findings to inline comments
  □ Use line numbers from evidence

□ Submit Review
  □ Call create_pull_request_review
  □ Return review URL
```

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| 422 Invalid | Line doesn't exist in diff | Use position instead of line |
| 404 Not Found | PR or commit doesn't exist | Verify PR number and commit SHA |
| 403 Forbidden | No permission to review | Check GitHub token permissions |

## Tips

1. **Commit ID**: Always use the head commit SHA from `retrieve-diff-from-github-pr`
2. **Line vs Position**: `line` refers to the line in the new file, `position` refers to the position in the diff hunk
3. **Batch Comments**: Submit all comments in one review to avoid notification spam
4. **Suggestion Blocks**: Use GitHub's suggestion syntax for easy one-click fixes

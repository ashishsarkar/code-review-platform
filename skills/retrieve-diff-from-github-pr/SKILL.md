---
name: retrieve-diff-from-github-pr
description: Retrieve code diff from a GitHub Pull Request via GitHub API. Use this as the first step in a code review pipeline when reviewing GitHub PRs.
metadata:
  author: Ashish
  version: "1.0"
  type: input
  requires: github-api
---

# Retrieve Diff from GitHub PR Skill

An **input skill** that retrieves code diff and PR metadata from GitHub via the API. This is the entry point for reviewing GitHub Pull Requests.

## Role

- **Fetch**: Get PR details and diff via GitHub API
- **Extract**: Parse changed files and their diffs
- **Context**: Gather PR metadata (title, description, author, reviews)

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| `owner` | Yes | Repository owner (username or organization) |
| `repo` | Yes | Repository name |
| `pull_number` | Yes | Pull Request number |

## Outputs

| Output | Description |
|--------|-------------|
| `pr_info` | PR metadata (title, description, author, state, labels) |
| `files` | List of changed files with patches |
| `commits` | List of commits in the PR |
| `base_ref` | Base branch reference |
| `head_ref` | Head branch reference |
| `diff` | Full unified diff content |

## Required MCP Tools

This skill uses the GitHub MCP server with the following tools:

| Tool | Purpose |
|------|---------|
| `get_pull_request` | Get PR details (title, body, state, author) |
| `get_pull_request_files` | Get list of changed files with patches |

## Step 1: Parse PR Reference

Accept PR reference in various formats:

| Format | Example | Parsing |
|--------|---------|---------|
| **Full URL** | `https://github.com/owner/repo/pull/123` | Extract owner, repo, pull_number |
| **Short reference** | `owner/repo#123` | Split on `/` and `#` |
| **Number only** | `123` | Requires owner/repo from context |

```
Regex for URL: github\.com/([^/]+)/([^/]+)/pull/(\d+)
Regex for short: ([^/]+)/([^#]+)#(\d+)
```

## Step 2: Fetch PR Details

Use the GitHub MCP tool to get PR information:

```json
{
  "tool": "get_pull_request",
  "server": "user-github",
  "arguments": {
    "owner": "<owner>",
    "repo": "<repo>",
    "pull_number": <number>
  }
}
```

This returns:
- PR title and description
- Author information
- Base and head branches
- State (open/closed/merged)
- Labels and reviewers
- Head commit SHA (needed for submitting review)

## Step 3: Fetch Changed Files

Use the GitHub MCP tool to get file changes:

```json
{
  "tool": "get_pull_request_files",
  "server": "user-github",
  "arguments": {
    "owner": "<owner>",
    "repo": "<repo>",
    "pull_number": <number>
  }
}
```

This returns for each file:
- `filename`: Path to the file
- `status`: added, removed, modified, renamed
- `additions`: Lines added
- `deletions`: Lines deleted
- `patch`: Unified diff patch for the file

## Step 4: Format Output

Structure the output for the review pipeline:

```markdown
## PR Retrieved

**Repository**: owner/repo
**PR #**: 123
**Title**: feat: add user authentication

### PR Info

| Field | Value |
|-------|-------|
| Author | @username |
| State | open |
| Base | main |
| Head | feature/auth |
| Commits | 3 |
| Changed Files | 5 |

### Description

<PR description content>

### Files Changed

| Status | File | +/- |
|--------|------|-----|
| modified | src/auth/login.ts | +50/-10 |
| added | src/utils/helper.ts | +30/-0 |
| deleted | src/deprecated.ts | +0/-25 |

### Diff Content

<unified diff for each file>
```

## Output Format

```json
{
  "source": "github-pr",
  "repository": {
    "owner": "owner",
    "repo": "repo"
  },
  "pull_request": {
    "number": 123,
    "title": "feat: add user authentication",
    "body": "PR description...",
    "author": "username",
    "state": "open",
    "base": {
      "ref": "main",
      "sha": "abc123"
    },
    "head": {
      "ref": "feature/auth",
      "sha": "def456"
    },
    "labels": ["enhancement"],
    "created_at": "2024-01-15T10:30:00Z",
    "updated_at": "2024-01-15T12:00:00Z"
  },
  "stats": {
    "files_changed": 5,
    "additions": 120,
    "deletions": 45,
    "commits": 3
  },
  "files": [
    {
      "path": "src/auth/login.ts",
      "status": "modified",
      "additions": 50,
      "deletions": 10,
      "patch": "@@ -1,10 +1,50 @@\n..."
    }
  ],
  "diff": "<combined unified diff>"
}
```

## Integration with Review Pipeline

After retrieving the PR diff, pass the output to:

1. **codereview-orchestrator** - For triage and routing
2. Then to appropriate specialist skills based on the triage
3. Finally to **submit-github-review** to post the review

## Quick Reference

```
□ Parse PR Reference
  □ Extract owner, repo, pull_number
  □ Handle URL or short format

□ Fetch PR Details
  □ Call get_pull_request MCP tool
  □ Extract metadata and head SHA

□ Fetch Changed Files
  □ Call get_pull_request_files MCP tool
  □ Get patches for all files

□ Format Output
  □ Structure for review pipeline
  □ Include commit SHA for review submission
```

## Example Usage in Pipeline

```
1. retrieve-diff-from-github-pr (this skill)
   ↓
2. codereview-orchestrator (triage)
   ↓
3. Specialist skills (review)
   ↓
4. submit-github-review (post review)
```

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| 404 Not Found | PR doesn't exist or no access | Verify PR number and permissions |
| 403 Forbidden | Rate limited or no auth | Check GitHub token |
| Invalid format | Can't parse PR reference | Use format: owner/repo#number |

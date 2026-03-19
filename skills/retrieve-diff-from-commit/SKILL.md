---
name: retrieve-diff-from-commit
description: Retrieve code diff from a local git commit. Use this as the first step in a code review pipeline when reviewing local commits.
metadata:
  author: Ashish
  version: "1.0"
  type: input
  requires: git
---

# Retrieve Diff from Commit Skill

An **input skill** that retrieves code diff from local git commits. This is the entry point for reviewing local changes before they are pushed.

## Role

- **Extract**: Get the diff content from a specific commit or range of commits
- **Format**: Prepare the diff in a format suitable for code review
- **Context**: Gather commit metadata (author, message, timestamp)

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| `commit_sha` | Optional | Specific commit SHA to review (defaults to HEAD) |
| `commit_range` | Optional | Range of commits (e.g., `main..HEAD`, `HEAD~3..HEAD`) |
| `base_branch` | Optional | Compare current branch against base (e.g., `main`, `develop`) |

## Outputs

| Output | Description |
|--------|-------------|
| `diff` | The unified diff content |
| `files_changed` | List of files modified with change type (added/modified/deleted) |
| `commit_info` | Commit metadata (SHA, author, message, timestamp) |
| `stats` | Summary statistics (lines added, deleted, files changed) |

## Usage

### Review Single Commit

```bash
# Get diff for specific commit
git show <commit_sha> --format="%H%n%an%n%ae%n%s%n%b" --stat

# Get unified diff
git diff <commit_sha>^..<commit_sha>
```

### Review Commit Range

```bash
# Get diff for range
git diff <base_commit>..<head_commit>

# Get list of commits in range
git log --oneline <base_commit>..<head_commit>
```

### Review Branch Against Base

```bash
# Compare feature branch to main
git diff main...HEAD

# Get merge-base
git merge-base main HEAD
```

## Step 1: Identify the Scope

Determine what to review:

| Scenario | Command | Use Case |
|----------|---------|----------|
| **Last commit** | `git diff HEAD~1..HEAD` | Quick review of latest change |
| **All uncommitted** | `git diff` | Review before committing |
| **Staged changes** | `git diff --cached` | Review what will be committed |
| **Branch diff** | `git diff main...HEAD` | Full feature review |
| **Specific commit** | `git show <sha>` | Review single commit |

## Step 2: Extract Diff

Execute the appropriate git command:

```bash
# For comprehensive diff with context
git diff --unified=5 <base>..<head>

# Include file stats
git diff --stat <base>..<head>

# Get list of changed files
git diff --name-status <base>..<head>
```

## Step 3: Gather Metadata

Collect commit information:

```bash
# Get commit details
git log --format="%H|%an|%ae|%at|%s" <base>..<head>

# Get file change summary
git diff --shortstat <base>..<head>
```

## Step 4: Format Output

Structure the output for the review pipeline:

```markdown
## Diff Retrieved

**Commit Range**: `<base>..<head>`
**Files Changed**: X
**Lines Added**: +Y
**Lines Deleted**: -Z

### Files

| Status | File |
|--------|------|
| M | src/auth/login.ts |
| A | src/utils/helper.ts |
| D | src/deprecated.ts |

### Diff Content

\`\`\`diff
<unified diff content>
\`\`\`

### Commit Info

| Field | Value |
|-------|-------|
| SHA | abc123... |
| Author | John Doe <john@example.com> |
| Date | 2024-01-15 10:30:00 |
| Message | feat: add login functionality |
```

## Output Format

```json
{
  "source": "local-git",
  "commit_range": {
    "base": "<base_sha>",
    "head": "<head_sha>"
  },
  "stats": {
    "files_changed": 5,
    "insertions": 120,
    "deletions": 45
  },
  "files": [
    {
      "path": "src/auth/login.ts",
      "status": "modified",
      "additions": 50,
      "deletions": 10
    }
  ],
  "commits": [
    {
      "sha": "abc123",
      "author": "John Doe",
      "email": "john@example.com",
      "timestamp": "2024-01-15T10:30:00Z",
      "message": "feat: add login functionality"
    }
  ],
  "diff": "<unified diff content>"
}
```

## Integration with Review Pipeline

After retrieving the diff, pass the output to:

1. **codereview-orchestrator** - For triage and routing
2. Then to appropriate specialist skills based on the triage

## Quick Reference

```
□ Identify Scope
  □ Single commit, range, or branch?
  □ Committed or uncommitted?

□ Extract Diff
  □ Run appropriate git diff command
  □ Include sufficient context (--unified=5)

□ Gather Metadata
  □ Commit info (SHA, author, message)
  □ File statistics

□ Format Output
  □ Structure for review pipeline
  □ Include all necessary context
```

## Example Usage in Pipeline

```
1. retrieve-diff-from-commit (this skill)
   ↓
2. codereview-orchestrator (triage)
   ↓
3. Specialist skills (review)
   ↓
4. Output formatted findings
```

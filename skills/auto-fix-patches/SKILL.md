---
name: auto-fix-patches
description: Transform review findings into concrete GitHub suggestion blocks that authors can accept with one click. Reads the original source code, generates exact replacement code, and formats it as GitHub-native suggestions in PR inline comments.
metadata:
  author: Ashish
  version: "1.0"
  type: output
  persona: Fix Generator
  requires: github-api
---

# Auto-fix Patches Skill

An **output skill** that converts review findings into **one-click GitHub suggestion blocks**. Instead of describing what to fix in prose, this skill reads the actual source code, generates the exact replacement, and posts it as a GitHub `suggestion` block that the PR author can apply with a single click.

## Role

- **Read**: Fetch the original source code at the lines referenced by findings
- **Generate**: Produce exact replacement code for each finding
- **Format**: Wrap fixes in GitHub `suggestion` blocks
- **Post**: Submit as inline comments on the PR

## What This Skill Does

- Turns text-based fix descriptions into concrete code replacements
- Produces GitHub-native suggestion blocks (one-click apply)
- Handles single-line and multi-line fixes
- Groups related fixes when they affect adjacent lines
- Preserves original indentation, style, and formatting

## What This Skill Does NOT Do

- Apply fixes automatically (the author decides)
- Push commits to the PR branch
- Modify the original review body
- Generate fixes for findings without enough context

## Trigger Conditions

Invoke this skill:
- After specialist skills produce findings with `fix` suggestions
- As a post-processing step before or alongside `submit-github-review`
- When the user says "Generate fix suggestions for PR #123"

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| `owner` | Yes | Repository owner |
| `repo` | Yes | Repository name |
| `pull_number` | Yes | Pull Request number |
| `commit_id` | Yes | HEAD commit SHA of the PR |
| `findings` | Yes | Array of findings from specialist skills |

## Outputs

| Output | Description |
|--------|-------------|
| `suggestions_generated` | Number of suggestion blocks created |
| `suggestions_skipped` | Findings where auto-fix wasn't possible |
| `comments_posted` | Number of inline comments posted to the PR |

## Step 1: Filter Fixable Findings

Not all findings can be auto-fixed. Filter for findings that have:

```yaml
fixable:
  - Has a `fix` field with actionable description
  - Has `evidence.file` and `evidence.line` pointing to specific code
  - The fix is a code change (not a process/architecture change)

not_fixable:
  - "Add tests for this function" → process change, skip
  - "Consider refactoring this module" → too broad, skip
  - "Review the auth flow" → no specific code fix, skip
  - Finding has no file/line reference → skip
```

### Fixability Classification

| Finding Type | Fixable? | Example |
|-------------|----------|---------|
| SQL injection | Yes | Replace string concat with parameterized query |
| Missing null check | Yes | Add guard clause |
| Unused import | Yes | Remove the import line |
| XSS vulnerability | Yes | Add output encoding |
| Missing error handling | Yes | Wrap in try/catch |
| Missing auth middleware | Partial | Can suggest decorator, but placement needs review |
| Architecture concern | No | "This should be a separate service" |
| Missing tests | No | Test generation is a different skill |
| Performance N+1 | Partial | Can suggest batch query, but needs context |

## Step 2: Fetch Original Source Code

For each fixable finding, fetch the actual source code around the affected lines:

```bash
# Get the file content at the PR's head commit
gh api repos/{owner}/{repo}/contents/{file_path}?ref={commit_id} \
  --jq '.content' | base64 -d
```

### Context Window

Fetch enough surrounding context to generate an accurate fix:

```yaml
context_lines:
  before: 5    # Lines before the affected line
  after: 5     # Lines after the affected line
  minimum: 1   # At least the affected line itself
```

This ensures:
- Correct indentation matching
- Awareness of surrounding code structure (open braces, etc.)
- Proper variable scoping

## Step 3: Generate Exact Replacement Code

For each finding, generate the replacement code:

### Rules for Code Generation

1. **Match the existing style exactly**
   - Same indentation (tabs vs spaces, indent level)
   - Same quote style (single vs double)
   - Same semicolon convention
   - Same bracket placement

2. **Minimal diff**
   - Only change what's necessary to fix the issue
   - Don't refactor surrounding code
   - Don't add unrelated improvements
   - Don't change formatting of untouched lines

3. **Correct and complete**
   - The suggestion must compile/parse on its own
   - Include any new imports needed (as a separate suggestion)
   - Don't leave TODO comments in suggestions

4. **Single responsibility**
   - One fix per suggestion block
   - If a fix requires changes in multiple places, create separate suggestions

### Examples

#### SQL Injection Fix

**Finding:**
```json
{
  "file": "src/users.ts",
  "line": 42,
  "fix": "Use parameterized query"
}
```

**Original code (line 42):**
```typescript
  const user = await db.query(`SELECT * FROM users WHERE id = ${userId}`);
```

**Generated suggestion:**
```markdown
```suggestion
  const user = await db.query('SELECT * FROM users WHERE id = ?', [userId]);
`` `
```

#### Missing Null Check

**Finding:**
```json
{
  "file": "src/api/handler.ts",
  "line": 15,
  "fix": "Add null check before accessing property"
}
```

**Original code (lines 15-16):**
```typescript
  const name = user.profile.name;
  return res.json({ name });
```

**Generated suggestion:**
```markdown
```suggestion
  const name = user?.profile?.name ?? 'Unknown';
  return res.json({ name });
`` `
```

#### Unused Import Removal

**Finding:**
```json
{
  "file": "src/utils.ts",
  "line": 3,
  "fix": "Remove unused import"
}
```

**Original code (line 3):**
```typescript
import { unusedHelper } from './helpers';
```

**Generated suggestion:**
```markdown
```suggestion
`` `
```
(Empty suggestion = delete the line)

## Step 4: Format as GitHub Suggestion Blocks

### Single-line Suggestion

For fixes that affect exactly one line:

```markdown
🔧 **Auto-fix** | <severity_emoji> **<category>**: <title>

<brief explanation of why this change is needed>

```suggestion
<replacement code for the single line>
`` `
```

### Multi-line Suggestion

For fixes that span multiple consecutive lines, use GitHub's multi-line suggestion syntax. The comment must be placed on the **last line** of the range, with `start_line` set to the first line:

```markdown
🔧 **Auto-fix** | <severity_emoji> **<category>**: <title>

<brief explanation>

```suggestion
<replacement code for all lines in the range>
`` `
```

### Suggestion with New Import

When a fix requires a new import, post two suggestions:

**Comment 1 (on the import section):**
```markdown
🔧 **Auto-fix** | Add required import

```suggestion
import { existingImport } from './module';
import { sanitize } from './security';
`` `
```

**Comment 2 (on the affected line):**
```markdown
🔧 **Auto-fix** | 🔴 **Security**: Sanitize user input

```suggestion
  const clean = sanitize(userInput);
  const result = await processData(clean);
`` `
```

## Step 5: Post Suggestions to GitHub

Use the GitHub API to post each suggestion as an inline comment:

### Single-line Comment

```bash
gh api repos/{owner}/{repo}/pulls/{pull_number}/comments \
  -f body='🔧 **Auto-fix** | 🔴 **Security**: SQL injection

Use parameterized queries to prevent SQL injection.

```suggestion
  const user = await db.query('\''SELECT * FROM users WHERE id = ?'\'', [userId]);
`` `' \
  -f commit_id='{commit_id}' \
  -f path='{file_path}' \
  -F line={line_number} \
  -f side='RIGHT'
```

### Multi-line Comment

```bash
gh api repos/{owner}/{repo}/pulls/{pull_number}/comments \
  -f body='🔧 **Auto-fix** | 🟡 **Correctness**: Add error handling

Wrap database call in try/catch to handle connection failures.

```suggestion
  try {
    const result = await db.query(sql);
    return result.rows;
  } catch (err) {
    logger.error('\''Query failed'\'', { error: err.message });
    throw new DatabaseError('\''Query failed'\'');
  }
`` `' \
  -f commit_id='{commit_id}' \
  -f path='{file_path}' \
  -F start_line={start_line} \
  -F line={end_line} \
  -f start_side='RIGHT' \
  -f side='RIGHT'
```

## Step 6: Handle Edge Cases

### Lines Not in Diff

GitHub only allows inline comments on lines that appear in the diff. If a finding references a line outside the diff:

- **Option A**: Post as a general PR comment with the file and line reference
- **Option B**: Skip the auto-fix, include in the review body summary instead

```markdown
💡 **Fix available but outside diff** — `src/utils.ts:142`

The fix for [issue] is on a line not modified in this PR. Apply manually:

\`\`\`diff
- const data = eval(input);
+ const data = JSON.parse(input);
\`\`\`
```

### Conflicting Fixes

When two findings affect the same line(s):

1. **If fixes are compatible**: Combine into one suggestion
2. **If fixes conflict**: Post both as separate comments, note the conflict

```markdown
⚠️ **Note**: This line has multiple suggested fixes. Apply one at a time and verify:

**Fix 1** (Security): Parameterize the query
```suggestion
const result = await db.query('SELECT * FROM users WHERE id = ?', [sanitize(userId)]);
`` `

**Fix 2** (Performance): Use a prepared statement
```suggestion
const result = await prepared.users.findById.execute([userId]);
`` `
```

### Indentation Detection

Detect the file's indentation style before generating suggestions:

```yaml
detect:
  - Tabs vs spaces (check first 20 lines)
  - Indent size (2, 4, or 8 spaces)
  - Trailing newline convention
  - Line ending style (LF vs CRLF)
```

## Output Format

```json
{
  "status": "success",
  "pr": "owner/repo#123",
  "suggestions": {
    "generated": 5,
    "posted": 4,
    "skipped": 1,
    "outside_diff": 1
  },
  "details": [
    {
      "finding_category": "security",
      "finding_severity": "blocker",
      "file": "src/users.ts",
      "line": 42,
      "type": "single_line",
      "status": "posted",
      "comment_url": "https://github.com/owner/repo/pull/123#discussion_r12345"
    },
    {
      "finding_category": "correctness",
      "finding_severity": "major",
      "file": "src/api/handler.ts",
      "lines": [15, 20],
      "type": "multi_line",
      "status": "posted",
      "comment_url": "https://github.com/owner/repo/pull/123#discussion_r12346"
    },
    {
      "finding_category": "architecture",
      "finding_severity": "minor",
      "file": "src/service.ts",
      "line": null,
      "type": "not_fixable",
      "status": "skipped",
      "reason": "Architecture suggestion - no specific code fix"
    }
  ]
}
```

## Pipeline Integration

This skill runs between specialist reviews and the final review submission:

```
1. retrieve-diff-from-github-pr
   ↓ (PR info + diff + commit_id)
2. codereview-orchestrator
   ↓ (triage + routing plan)
3. Specialist skills (parallel or sequential)
   ↓ (findings array)
4. auto-fix-patches (this skill)
   ↓ (suggestion comments posted)
5. submit-github-review
   ↓ (review summary posted)
6. interactive-chat
   ↓ (handle replies)
```

### Integration with submit-github-review

When `auto-fix-patches` runs first:
- Findings that got suggestion blocks are marked as "fix suggested"
- `submit-github-review` references them in the review body: "🔧 Fix suggested — see inline comment"
- Avoids duplicate inline comments for the same finding

## Quick Reference

```
□ Filter Findings
  □ Has file + line reference?
  □ Has actionable fix description?
  □ Is a code change (not process)?

□ Fetch Source
  □ Get file at PR head commit
  □ Include surrounding context (±5 lines)

□ Generate Fix
  □ Match existing code style exactly
  □ Minimal diff - only change what's needed
  □ Fix must be complete and correct
  □ One fix per suggestion block

□ Format Suggestion
  □ Single-line → standard suggestion block
  □ Multi-line → start_line + line range
  □ New import needed → separate suggestion

□ Post to GitHub
  □ Inline comment with suggestion block
  □ Handle lines outside diff gracefully
  □ Handle conflicting fixes on same line

□ Report
  □ Count generated / posted / skipped
  □ Note any outside-diff fixes
```

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| 422 line not in diff | Fix is on unchanged line | Post as general comment with diff block |
| 422 invalid suggestion | Suggestion syntax error | Validate suggestion block formatting |
| 404 file not found | File was deleted/renamed | Skip, note in output |
| Fix changes semantics | Generated code is incorrect | Validate fix preserves function signatures and types |

## Tips

1. **Validate before posting** - Ensure the suggestion block syntax is correct (matching backticks, no nested code blocks)
2. **One click = one fix** - Each suggestion should be independently applicable
3. **Preserve context** - Multi-line suggestions must include ALL lines in the range, even unchanged ones
4. **Style matching** - The #1 reason authors reject suggestions is mismatched style
5. **Be conservative** - When in doubt about the correct fix, describe it in text instead of a suggestion block
6. **Test mentally** - Before posting, mentally apply the suggestion and verify it compiles

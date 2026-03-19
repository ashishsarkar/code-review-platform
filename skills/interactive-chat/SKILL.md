---
name: interactive-chat
description: Respond to PR comment threads on GitHub. When a PR author or reviewer replies to a review comment, this skill fetches the conversation context, understands the question or pushback, and posts a helpful follow-up reply. Enables back-and-forth discussion on review findings.
metadata:
  author: Ashish
  version: "1.0"
  type: interaction
  persona: Collaborative Reviewer
  requires: github-api
---

# Interactive Chat Skill

An **interaction skill** that enables back-and-forth conversation on GitHub PR review comments. When a PR author questions a finding, asks for clarification, or pushes back, this skill understands the context and replies helpfully.

## Role

- **Listen**: Detect and fetch new replies on review comments
- **Understand**: Parse the author's question, pushback, or request
- **Respond**: Post a context-aware reply to the comment thread
- **Adapt**: Update or retract findings when the author provides valid reasoning

## What This Skill Does

- Reply to questions about review findings ("Why is this a problem?")
- Acknowledge valid pushback and retract false positives
- Provide deeper explanations with code examples
- Suggest alternative fixes when the original suggestion doesn't work
- Answer general questions about the code in the PR context

## What This Skill Does NOT Do

- Perform new code review (delegate to specialists)
- Approve or request changes (delegate to submit-github-review)
- Modify the original review body

## Trigger Conditions

Invoke this skill when:
- A user says "Reply to comment on PR #123"
- A user says "Check for new comments on PR #123"
- A user says "Respond to @author on PR #123"
- A notification arrives about a reply to a review comment

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| `owner` | Yes | Repository owner |
| `repo` | Yes | Repository name |
| `pull_number` | Yes | Pull Request number |
| `comment_id` | Optional | Specific comment to respond to (if omitted, scan all threads) |
| `review_id` | Optional | Specific review to scope the conversation to |

## Outputs

| Output | Description |
|--------|-------------|
| `replies_posted` | Number of replies posted |
| `threads_resolved` | Threads where the concern was addressed |
| `findings_retracted` | Findings acknowledged as false positives |
| `threads_skipped` | Threads that don't need a response |

## Step 1: Fetch Comment Threads

Retrieve all review comment threads on the PR:

```bash
# Fetch all review comments on a PR
gh api repos/{owner}/{repo}/pulls/{pull_number}/comments

# Fetch replies to a specific comment thread
gh api repos/{owner}/{repo}/pulls/comments/{comment_id}/replies

# Fetch general issue comments (non-inline)
gh api repos/{owner}/{repo}/issues/{pull_number}/comments
```

### Identify Actionable Threads

A thread needs a response when:
- The **last reply** is from someone other than the bot
- The reply contains a question, disagreement, or request
- The thread is not yet resolved

Skip threads where:
- The bot already has the last reply
- The reply is just "thanks" or an acknowledgement
- The thread has been resolved by the author

## Step 2: Understand the Reply

Classify the author's reply into one of these intents:

| Intent | Signals | Example |
|--------|---------|---------|
| **Question** | "Why?", "Can you explain?", "What do you mean?" | "Why is this a security issue?" |
| **Pushback** | "This is intentional", "We handle this elsewhere", "False positive" | "We sanitize input in the middleware layer" |
| **Clarification** | "Did you mean X or Y?", "Which line?" | "Are you referring to the query on line 42 or 58?" |
| **Alternative** | "What about doing X instead?", "Would Y also work?" | "Can we use a prepared statement here instead?" |
| **Acceptance** | "Good catch", "Will fix", "Makes sense" | "Thanks, I'll update this" |
| **More Info** | "Here's the context...", "The reason we did this..." | "This endpoint is internal-only behind a VPN" |

## Step 3: Gather Context

Before responding, gather the necessary context:

```yaml
context:
  # Always fetch
  - original_review_comment: The initial finding that started the thread
  - reply_chain: All replies in the thread in order
  - diff_hunk: The code being discussed

  # Fetch if needed
  - full_file: When the discussion references code outside the diff
  - related_files: When the author mentions handling something "elsewhere"
  - pr_description: When the author references the PR intent
```

### Context Gathering Commands

```bash
# Get the specific review comment with diff context
gh api repos/{owner}/{repo}/pulls/comments/{comment_id}

# Get the full file at the PR head commit to check broader context
gh api repos/{owner}/{repo}/contents/{path}?ref={head_sha}

# Search for related patterns the author mentions
# (e.g., "we sanitize in middleware" -> find the middleware)
gh api -X GET "repos/{owner}/{repo}/search/code?q=sanitize+repo:{owner}/{repo}"
```

## Step 4: Generate Response

### Response Guidelines

1. **Be conversational, not robotic** - Write like a helpful colleague, not a linter
2. **Acknowledge valid points** - If the author is right, say so clearly
3. **Show evidence** - Link to code, docs, or standards when making a claim
4. **Be concise** - Keep replies focused; don't re-explain the entire finding
5. **Ask follow-ups** - If uncertain, ask a clarifying question rather than guessing
6. **Stay in scope** - Only discuss the specific issue in the thread

### Response Templates by Intent

#### Responding to a Question

```markdown
Good question! The concern here is [specific risk].

[1-2 sentence explanation with concrete example of what could go wrong]

For example:
```<language>
// An attacker could...
<exploit example>
```

[Link to relevant documentation or standard if applicable]
```

#### Responding to Pushback (Valid)

```markdown
You're right - I missed that [the mitigation the author described]. Since [reason it's safe], this isn't an issue.

Retracted - thanks for the context!
```

#### Responding to Pushback (Still Concerned)

```markdown
I see the [mitigation they described], but the concern is slightly different here:

[Explain why the mitigation doesn't fully address the issue]

Specifically, [concrete scenario where the issue still applies].

Would [alternative approach] work for your use case?
```

#### Responding to a Request for Alternatives

```markdown
Absolutely, here are a couple of options:

**Option A**: [approach]
```<language>
<code example>
```

**Option B**: [approach]
```<language>
<code example>
```

Option A is [tradeoff], while Option B is [tradeoff]. Given [their context], I'd lean toward [recommendation].
```

#### Responding to Acceptance

No reply needed. Optionally mark thread as resolved.

#### Responding to More Info

```markdown
Thanks for the context! Given that [restate their point], [updated assessment].

[If it changes the finding]: I'll retract this finding - the [mitigation] addresses the concern.

[If it doesn't change the finding]: Even with [their context], I'd still recommend [action] because [reason].
```

## Step 5: Post the Reply

```bash
# Reply to a review comment thread
gh api repos/{owner}/{repo}/pulls/{pull_number}/comments \
  -f body="<response>" \
  -F in_reply_to=<parent_comment_id>

# Reply to a general issue comment
gh api repos/{owner}/{repo}/issues/{pull_number}/comments \
  -f body="<response>"
```

## Step 6: Track State

After responding, track the outcome:

```json
{
  "thread_id": 12345,
  "original_finding": {
    "severity": "major",
    "category": "security",
    "description": "SQL injection vulnerability"
  },
  "author_intent": "pushback",
  "action_taken": "retracted",
  "reason": "Input is sanitized in middleware layer - verified at src/middleware/sanitize.ts:15",
  "reply_posted": true
}
```

### Retraction Policy

Retract a finding when the author demonstrates:
- The issue is handled elsewhere in the codebase (and you verified it)
- The flagged pattern is safe in this specific context (e.g., internal-only endpoint)
- The finding is based on a misunderstanding of the code's purpose

Do NOT retract when:
- The author says "it's fine" without evidence
- The mitigation is partial or unreliable
- The risk is accepted but not mitigated

## Output Format

```json
{
  "status": "success",
  "pr": "owner/repo#123",
  "threads_processed": 5,
  "replies_posted": 3,
  "threads_skipped": 2,
  "details": [
    {
      "thread_id": 12345,
      "intent": "question",
      "action": "explained",
      "reply_url": "https://github.com/owner/repo/pull/123#discussion_r12345"
    },
    {
      "thread_id": 12346,
      "intent": "pushback",
      "action": "retracted",
      "reason": "Verified mitigation in middleware",
      "reply_url": "https://github.com/owner/repo/pull/123#discussion_r12346"
    },
    {
      "thread_id": 12347,
      "intent": "acceptance",
      "action": "skipped",
      "reason": "Author acknowledged - no reply needed"
    }
  ]
}
```

## Pipeline Integration

This skill operates **after** the initial review is submitted and runs on-demand or on notification:

```
1. retrieve-diff-from-github-pr
   ↓
2. codereview-orchestrator → specialists
   ↓
3. submit-github-review
   ↓ (review posted, author replies)
4. interactive-chat (this skill)  ←→  Author
   ↓ (loop until threads resolved)
5. Done
```

## Quick Reference

```
□ Fetch Threads
  □ Get all review comment threads
  □ Identify threads needing response
  □ Skip resolved / acknowledged threads

□ Understand Intent
  □ Question → Explain the finding
  □ Pushback → Verify claim, retract or defend
  □ Clarification → Provide specifics
  □ Alternative → Evaluate and recommend
  □ Acceptance → Skip or resolve
  □ More Info → Reassess finding

□ Gather Context
  □ Original finding + reply chain
  □ Diff hunk + full file if needed
  □ Related files if author references them

□ Respond
  □ Conversational tone
  □ Show evidence
  □ Be concise
  □ Acknowledge valid points

□ Post Reply
  □ Use in_reply_to for threading
  □ Track retracted findings
```

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| 404 Not Found | Comment was deleted | Skip thread, log as deleted |
| 403 Forbidden | No permission to comment | Check GitHub token permissions |
| 422 Unprocessable | Reply to locked conversation | Skip thread, note it's locked |
| Rate Limited | Too many API calls | Back off and retry with delay |

## Tips

1. **Don't be defensive** - If a finding is wrong, retract it gracefully
2. **Verify before retracting** - Don't just take the author's word; check the code they reference
3. **One reply per thread** - Don't post multiple replies in the same thread at once
4. **Quote the author** - Use `> their words` to show you read their reply carefully
5. **Link to code** - Reference specific files and lines when discussing context
6. **Know when to stop** - If the discussion goes in circles, suggest resolving offline

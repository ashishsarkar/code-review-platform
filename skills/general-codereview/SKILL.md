---
name: general-codereview
description: Perform thorough code reviews following Google's 5-step methodology. Use when reviewing pull requests, change lists, or any code changes to ensure quality, correctness, and maintainability.
license: MIT
metadata:
  author: Ashish
  version: "1.0"
  origin: "Based on 5 steps to perform code reviews at Google (2013)"
---

# General Code Review Skill

A comprehensive code review methodology based on Google's engineering practices. Follow these 5 steps to perform effective and thorough code reviews.

## Step 1: Pre-screen

Before diving into the code, perform a quick pre-screening:

### Check Size

- **Reasonable size**: A change list of ~300 lines of code is typically reasonable
- **Too large**: Larger CLs should be strongly discouraged unless necessary
  - Leave a comment asking if the CL can be broken down into smaller, logically complete parts
  - Ask for the reason if breaking down is not possible
- **Small is OK**: Even single-line changes are fine as long as they are logically complete

### Verify You're the Right Reviewer

Ask yourself:
- Am I familiar enough with this code area?
- Do I understand the language, environment, and context?
- Is there someone better suited to review this?

If you're not the ideal reviewer but no one else is available, prepare to quickly learn about the unfamiliar aspects.

## Step 2: Understand

After pre-screening, focus on understanding the change from the author's perspective.

### Purpose

Ask: **"Do I know the overall purpose of this CL?"**

- The purpose should be clearly stated in the CL description
- If unclear, request clarification
- Even self-explanatory code needs a clear description for future maintainers
- Remember: even the author may need to refer back to the description later

### Workflow Context

Ask: **"Was I expecting this CL? Does it match my expectations?"**

Consider:
- Is this CL part of a larger project or design doc?
- Does it align with the task distribution and design documentation?
- Are there dependencies mentioned?
  - Other changes that need to happen before this CL can function
  - Other CLs that are blocked by this change

### Design Decisions

Verify that important design decisions are:
- Documented in the code description
- Explained with rationale when non-obvious

## Step 3: Verify

Now verify the implementation matches the stated intent.

### Code-Description Alignment

- **Verify the code does what the description says**
- Flag any discrepancies between stated purpose and actual implementation

### Test Coverage

Verify relevant tests exist for:

| Category | What to Check |
|----------|---------------|
| Design decisions | Tests that validate key architectural choices |
| Important defaults | Tests for default values and their behavior |
| Complex logic | Tests for intricate algorithms or business logic |
| Edge cases | Zero, null, empty, boundary conditions, if-statement edges |

## Step 4: Optimize

Look for opportunities to improve the code's efficiency and maintainability.

### Logic Optimization

- Can the algorithm be simplified?
- Are there unnecessary computations?
- Is the control flow optimal?

### Library Usage

- Is the best library being used for this task?
- Are there more appropriate or efficient alternatives?
- Is the library being used correctly and efficiently?

## Step 5: Check Style

Final pass for code style and conventions.

### Style Checklist

- [ ] Follows project/team coding standards
- [ ] Consistent naming conventions
- [ ] Proper formatting and indentation
- [ ] Clear and helpful comments
- [ ] No dead code or debugging artifacts

---

## Quick Reference Checklist

```
□ Pre-screen
  □ Size reasonable (~300 lines)?
  □ Am I the right reviewer?

□ Understand
  □ Purpose clear in description?
  □ Expected in current workflow?
  □ Design decisions documented?

□ Verify
  □ Code matches description?
  □ Tests cover design decisions?
  □ Tests cover defaults and edge cases?

□ Optimize
  □ Logic optimizable?
  □ Library choices optimal?

□ Check
  □ Style guidelines followed?
```

---

*Acknowledgment: This methodology is based on reflections from working at Google, with thanks to mentors and teammates who helped improve these code review skills.*

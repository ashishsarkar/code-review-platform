---
name: codereview-correctness
description: Analyze code for logic bugs, error handling issues, and edge cases. Detects off-by-one errors, null handling, race conditions, and incorrect error paths. Use when reviewing core business logic or complex algorithms.
metadata:
  author: Ashish
  version: "1.0"
  persona: Logic Bug Hunter
---

# Code Review Correctness Skill

A specialist focused on finding logic bugs, error handling issues, and edge case failures. This skill thinks about what can go wrong at runtime.

## Role

- **Bug Detection**: Find logic errors before they hit production
- **Edge Case Analysis**: Identify unhandled scenarios
- **Error Path Verification**: Ensure errors are handled correctly

## Persona

You are a senior engineer who has debugged thousands of production incidents. You know that most bugs come from assumptions that don't hold, edge cases that weren't considered, and error paths that weren't tested.

## Checklist

### Logic Bugs

- [ ] **Off-by-One Errors**: Array bounds, loop limits, string slicing
  ```javascript
  // 🚨 Off-by-one
  for (let i = 0; i <= arr.length; i++)  // should be <
  
  // 🚨 Fence-post error
  const pages = total / pageSize  // should be Math.ceil()
  ```

- [ ] **Wrong Conditions**: Inverted logic, wrong operators
  ```javascript
  // 🚨 Wrong operator
  if (status = "active")  // should be ===
  
  // 🚨 Inverted logic
  if (!isValid || !isEnabled)  // should this be &&?
  ```

- [ ] **Null/Undefined Handling**: Missing null checks
  ```javascript
  // 🚨 Potential null dereference
  const name = user.profile.name  // user or profile could be null
  
  // ✅ Safe access
  const name = user?.profile?.name ?? 'Unknown'
  ```

- [ ] **Type Coercion Bugs**: Implicit conversions causing issues
  ```javascript
  // 🚨 String + number = string
  const result = "5" + 3  // "53" not 8
  
  // 🚨 Truthy/falsy confusion
  if (count)  // 0 is falsy but may be valid
  ```

### Race Conditions & Ordering

- [ ] **TOCTOU (Time-of-Check-Time-of-Use)**:
  ```javascript
  // 🚨 Race condition
  if (await fileExists(path)) {
    await readFile(path)  // file might be deleted between check and read
  }
  ```

- [ ] **Ordering Assumptions**: Assuming operations complete in order
  ```javascript
  // 🚨 No ordering guarantee
  users.forEach(async user => await process(user))
  // The loop completes before any process() finishes
  ```

- [ ] **Shared State Modification**: Multiple paths modifying same state
  ```javascript
  // 🚨 Race on shared state
  if (!cache[key]) {
    cache[key] = await expensiveCompute()  // multiple calls may compute
  }
  ```

### Error Handling

- [ ] **Swallowed Errors**: Catch blocks that don't handle or rethrow
  ```javascript
  // 🚨 Error swallowed
  try { riskyOperation() } 
  catch (e) { console.log(e) }  // then what?
  
  // ✅ Proper handling
  try { riskyOperation() }
  catch (e) { 
    logger.error('Operation failed', { error: e })
    throw new OperationError('Failed', { cause: e })
  }
  ```

- [ ] **Wrong Error Type Caught**: Catching too broadly
  ```javascript
  // 🚨 Catches everything including programming errors
  try { ... }
  catch (e) { return defaultValue }  // hides bugs
  
  // ✅ Specific error handling
  catch (e) {
    if (e instanceof NotFoundError) return defaultValue
    throw e  // rethrow unexpected errors
  }
  ```

- [ ] **Missing Finally**: Resources not cleaned up on error
  ```javascript
  // 🚨 Connection leak on error
  const conn = await getConnection()
  await query(conn)  // if this throws, conn is never released
  
  // ✅ Always cleanup
  try { await query(conn) }
  finally { conn.release() }
  ```

- [ ] **Error Propagation**: Errors lost in async chains
  ```javascript
  // 🚨 Error lost
  promise.then(handleSuccess)  // no .catch()
  
  // 🚨 Error in event handler
  emitter.on('data', async (d) => await process(d))  // unhandled rejection
  ```

### Edge Cases

- [ ] **Empty Input**: What happens with `[]`, `""`, `null`, `undefined`?
  ```javascript
  // 🚨 Crashes on empty array
  const first = items[0].name  
  
  // ✅ Handles empty
  const first = items[0]?.name ?? 'default'
  ```

- [ ] **Huge Input**: What happens with 1M records?
  - Memory exhaustion?
  - Timeout?
  - Stack overflow (recursion)?

- [ ] **Unexpected Types**: What if wrong type is passed?
  ```javascript
  // 🚨 No type validation
  function process(id) {
    return id.toString()  // fails if id is null
  }
  ```

- [ ] **Boundary Values**: Min, max, zero, negative
  ```javascript
  // 🚨 Negative index
  const item = arr[index]  // what if index is -1?
  
  // 🚨 Integer overflow (in some languages)
  const total = price * quantity  // can this overflow?
  ```

### Production Assumptions

- [ ] **Clock/Timezone Issues**:
  ```javascript
  // 🚨 Timezone-naive
  const today = new Date().toISOString().split('T')[0]
  
  // 🚨 Midnight crossing
  if (startDate === endDate)  // what about times?
  ```

- [ ] **Locale/i18n Issues**:
  ```javascript
  // 🚨 Locale-dependent
  const lower = str.toLowerCase()  // Turkish 'I' problem
  parseFloat("1,234.56")  // fails in European locales
  ```

- [ ] **Floating Point**:
  ```javascript
  // 🚨 Floating point comparison
  if (0.1 + 0.2 === 0.3)  // false!
  
  // 🚨 Currency calculation
  const total = 19.99 * 100  // 1998.9999999999998
  ```

- [ ] **Retry/Idempotency**:
  - What if this operation runs twice?
  - What if it's retried after partial completion?

## Output Format

```json
{
  "findings": [
    {
      "severity": "major",
      "category": "correctness",
      "type": "null-dereference",
      "evidence": {
        "file": "src/users.ts",
        "line": 42,
        "snippet": "const name = user.profile.name"
      },
      "impact": "Crashes if user has no profile",
      "fix": "Use optional chaining: user?.profile?.name ?? 'Unknown'",
      "test": "Test with user object that has null profile"
    }
  ]
}
```

## Quick Reference

```
□ Logic Bugs
  □ Off-by-one errors?
  □ Wrong conditions/operators?
  □ Null/undefined handled?
  □ Type coercion issues?

□ Race Conditions
  □ TOCTOU vulnerabilities?
  □ Ordering guaranteed?
  □ Shared state protected?

□ Error Handling
  □ Errors not swallowed?
  □ Right errors caught?
  □ Resources cleaned up?
  □ Async errors propagated?

□ Edge Cases
  □ Empty input handled?
  □ Huge input considered?
  □ Wrong types rejected?
  □ Boundaries validated?

□ Production
  □ Timezone aware?
  □ Locale independent?
  □ Float precision handled?
  □ Idempotent operations?
```

## Common Bug Patterns by Severity

### Blockers 🔴
- Null dereference in critical path
- Infinite loops
- Data corruption potential

### Major 🟠
- Race conditions with data inconsistency
- Error handling that loses data
- Edge cases that cause silent failures

### Minor 🟡
- Inefficient error handling patterns
- Missing validation on internal APIs
- Overly broad exception catching

### Nits 💭
- Could use optional chaining
- Verbose null checks
- Redundant type checks

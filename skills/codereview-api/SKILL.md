---
name: codereview-api
description: Review API contracts, breaking changes, and interface consistency. Analyzes REST/RPC endpoints, event schemas, versioning, and backward compatibility. Use when reviewing public interfaces, API routes, or service contracts.
metadata:
  author: Ashish
  version: "1.0"
  persona: API Contract Guardian
---

# Code Review API Skill

A specialist focused on API contracts, breaking changes, and interface consistency. This skill ensures APIs remain stable, well-designed, and backward compatible.

## Role

- **Contract Enforcement**: Ensure APIs maintain their promises
- **Breaking Change Detection**: Identify changes that break consumers
- **Design Review**: Validate API design patterns

## Persona

You are an API platform engineer responsible for maintaining contracts with hundreds of consumers. You know that breaking changes cause cascading failures and that good API design prevents years of technical debt.

## Checklist

### Breaking Changes

- [ ] **Removed Endpoints**: Any endpoint deleted or moved?
  ```diff
  - DELETE /api/v1/users/:id/legacy
  ```

- [ ] **Changed Response Structure**: Fields removed, renamed, or type changed?
  ```diff
  {
  -  "user_id": "123",
  +  "userId": "123",      // renamed
  -  "count": "42",
  +  "count": 42,          // type changed
  -  "metadata": {...}     // removed
  }
  ```

- [ ] **Changed Request Parameters**: Required params added, optional made required?
  ```diff
  - POST /users { name }
  + POST /users { name, email }  // email now required = breaking
  ```

- [ ] **Changed HTTP Methods**: GET → POST, etc.?

- [ ] **Changed Status Codes**: 200 → 201, 400 → 422?

### Backward Compatibility

- [ ] **Additive Changes Only**: New fields should be optional
  ```javascript
  // ✅ Safe: new optional field
  { name, email, phone? }
  
  // 🚨 Breaking: new required field
  { name, email, phone }  // old clients don't send phone
  ```

- [ ] **Deprecation Path**: Is old behavior still supported?
  ```javascript
  // ✅ Deprecation with timeline
  /**
   * @deprecated Use /v2/users instead. Removal: 2025-01-01
   */
  ```

- [ ] **Version Strategy**: Is versioning consistent?
  - URL versioning: `/v1/`, `/v2/`
  - Header versioning: `Accept: application/vnd.api+json;version=2`
  - Query param: `?version=2`

### Input Validation

- [ ] **All Inputs Validated**: Type, range, format, length
  ```javascript
  // 🚨 No validation
  const user = await createUser(req.body)
  
  // ✅ Validated
  const data = validateSchema(req.body, CreateUserSchema)
  const user = await createUser(data)
  ```

- [ ] **Validation Errors Clear**: Error messages help the caller
  ```javascript
  // 🚨 Unhelpful
  { error: "Validation failed" }
  
  // ✅ Actionable
  { 
    error: "Validation failed",
    details: [
      { field: "email", message: "Invalid email format" },
      { field: "age", message: "Must be between 0 and 150" }
    ]
  }
  ```

- [ ] **Reject Unknown Fields**: Or explicitly allow them
  ```javascript
  // ✅ Explicit about extra fields
  validateSchema(data, schema, { stripUnknown: true })
  ```

### Output Guarantees

- [ ] **Consistent Response Shape**: Same shape for success/error
  ```javascript
  // ✅ Consistent envelope
  { success: true, data: {...} }
  { success: false, error: {...} }
  ```

- [ ] **Null vs Absent**: Consistent handling
  ```javascript
  // 🚨 Inconsistent
  { name: "Alice", bio: null }  // vs { name: "Bob" } (bio absent)
  
  // ✅ Consistent: always include, use null for missing
  { name: "Bob", bio: null }
  ```

- [ ] **Pagination Consistent**: Same structure across endpoints
  ```javascript
  // ✅ Standard pagination
  {
    data: [...],
    pagination: {
      page: 1,
      pageSize: 20,
      total: 100,
      hasMore: true
    }
  }
  ```

### Idempotency

- [ ] **Idempotent Methods**: GET, PUT, DELETE should be idempotent
  ```javascript
  // 🚨 Non-idempotent PUT
  PUT /counter  // increments each time
  
  // ✅ Idempotent PUT
  PUT /counter { value: 5 }  // sets to 5, repeatable
  ```

- [ ] **Idempotency Keys**: For non-idempotent operations
  ```javascript
  // ✅ Idempotency key for payments
  POST /payments
  Idempotency-Key: abc-123
  ```

- [ ] **Safe Retries**: What happens if called twice?

### Error Codes & Messages

- [ ] **Stable Error Codes**: Clients can programmatically handle
  ```javascript
  // ✅ Stable code
  { code: "USER_NOT_FOUND", message: "..." }
  
  // 🚨 Unstable: message can change
  { error: "User with ID 123 not found" }
  ```

- [ ] **Appropriate HTTP Status Codes**:
  | Scenario | Code |
  |----------|------|
  | Success | 200, 201, 204 |
  | Client error | 400, 401, 403, 404, 422 |
  | Server error | 500, 502, 503 |

- [ ] **No Stack Traces in Production**: Error details for devs, not users

### Documentation

- [ ] **API Docs Updated**: OpenAPI/Swagger, README
- [ ] **Examples Accurate**: Request/response examples work
- [ ] **Changelog Entry**: For significant changes

## Output Format

```markdown
## API Review Findings

### Breaking Changes 🔴

| Change | Impact | Migration Path |
|--------|--------|----------------|
| Removed `user_id` field | All clients using this field | Use `id` instead |
| Changed `/users` to require `email` | Clients not sending email | Add email to requests |

### Compatibility Issues 🟡

| Issue | Location | Recommendation |
|-------|----------|----------------|
| New required field | `POST /orders` | Make `shipping_address` optional |
| Missing deprecation notice | `GET /v1/legacy` | Add deprecation header |

### Design Recommendations 💡

- Consider pagination for `/users` endpoint (currently returns all)
- Add rate limiting headers to responses
- Include request ID in error responses
```

## Quick Reference

```
□ Breaking Changes
  □ Endpoints removed/moved?
  □ Response fields removed?
  □ Request params now required?
  □ Types changed?

□ Backward Compatibility
  □ New fields optional?
  □ Deprecation path exists?
  □ Versioning consistent?

□ Validation
  □ Inputs validated?
  □ Errors actionable?
  □ Unknown fields handled?

□ Output Guarantees
  □ Response shape consistent?
  □ Null handling consistent?
  □ Pagination standard?

□ Idempotency
  □ Methods correctly idempotent?
  □ Idempotency keys for mutations?

□ Errors
  □ Codes stable?
  □ Status codes appropriate?
  □ No internal details leaked?
```

## API Design Principles

### REST Best Practices
- Use nouns for resources: `/users`, not `/getUsers`
- Use HTTP methods correctly: GET reads, POST creates, PUT replaces, PATCH updates, DELETE removes
- Use plural nouns: `/users`, not `/user`
- Nest for relationships: `/users/:id/orders`

### Common Anti-Patterns
- God endpoints: `POST /api/doEverything`
- Verbs in URLs: `/api/createUser`
- Inconsistent naming: mixing `user_id` and `userId`
- Overloaded POST: using POST for everything

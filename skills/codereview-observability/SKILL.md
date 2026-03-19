---
name: codereview-observability
description: Review logging, metrics, tracing, and alerting. Ensures systems are observable, debuggable, and operable. Use when reviewing code that adds logging, metrics, or monitoring.
metadata:
  author: Ashish
  version: "1.0"
  persona: SRE/Observability Engineer
---

# Code Review Observability Skill

A specialist focused on logging, metrics, tracing, and alerting. This skill ensures systems can be monitored, debugged, and operated effectively.

## Role

- **Logging Quality**: Ensure logs are useful, not noisy
- **Metrics Coverage**: Verify key signals are captured
- **Tracing**: Ensure distributed operations can be traced
- **Alertability**: Check that failures are detectable

## Persona

You are an SRE who gets paged at 3 AM. You know that good observability is the difference between a 5-minute fix and a 5-hour investigation. You've seen logs that tell you nothing and dashboards that lie.

## Checklist

### Logging

- [ ] **Not Too Chatty**: Logs provide signal, not noise
  ```javascript
  // 🚨 Too verbose - drowns important logs
  items.forEach(item => logger.info('Processing item', item))
  
  // ✅ Appropriate granularity
  logger.info('Processing batch', { count: items.length })
  items.forEach(item => logger.debug('Processing item', { id: item.id }))
  ```

- [ ] **Correlation IDs**: Can trace request through system
  ```javascript
  // ✅ Request ID propagated
  logger.info('Processing order', { 
    requestId: ctx.requestId,
    orderId: order.id 
  })
  ```

- [ ] **No Secrets in Logs**: PII and secrets masked
  ```javascript
  // 🚨 Secrets in logs
  logger.info('Auth attempt', { email, password })
  
  // ✅ Secrets masked
  logger.info('Auth attempt', { email, password: '[REDACTED]' })
  ```

- [ ] **Structured Logging**: Logs are parseable
  ```javascript
  // 🚨 Unstructured
  logger.info(`User ${userId} created order ${orderId}`)
  
  // ✅ Structured
  logger.info('Order created', { userId, orderId })
  ```

- [ ] **Log Levels Appropriate**:
  | Level | Use For |
  |-------|---------|
  | ERROR | Failures requiring attention |
  | WARN | Potential issues, degraded state |
  | INFO | Significant business events |
  | DEBUG | Detailed diagnostic info |

- [ ] **Error Context**: Errors include enough context
  ```javascript
  // 🚨 No context
  logger.error('Failed')
  
  // ✅ Rich context
  logger.error('Order processing failed', {
    orderId,
    step: 'payment',
    error: e.message,
    stack: e.stack
  })
  ```

### Metrics

- [ ] **Success/Failure Counters**: Know if things work
  ```javascript
  // ✅ Counter for outcomes
  metrics.increment('orders.created', { status: 'success' })
  metrics.increment('orders.created', { status: 'failure', reason })
  ```

- [ ] **Latency Histograms**: Know how fast things are
  ```javascript
  // ✅ Histogram for latency
  const start = Date.now()
  await processOrder()
  metrics.histogram('order.processing.duration', Date.now() - start)
  ```

- [ ] **Saturation Metrics**: Know when at capacity
  ```javascript
  // ✅ Queue depth, connection pool usage
  metrics.gauge('queue.depth', queue.length)
  metrics.gauge('db.connections.active', pool.active)
  metrics.gauge('db.connections.idle', pool.idle)
  ```

- [ ] **Rate Metrics**: Know throughput
  ```javascript
  // ✅ Requests per second
  metrics.increment('http.requests', { method, path, status })
  ```

- [ ] **Cardinality Reasonable**: Labels don't explode
  ```javascript
  // 🚨 High cardinality
  metrics.increment('request', { userId })  // millions of users = millions of series
  
  // ✅ Bounded cardinality
  metrics.increment('request', { userType })  // few user types
  ```

### Tracing

- [ ] **Spans Around External Calls**: Can see dependencies
  ```javascript
  // ✅ Span for external call
  const span = tracer.startSpan('db.query')
  try {
    const result = await db.query(sql)
    span.setTag('rows', result.length)
    return result
  } finally {
    span.finish()
  }
  ```

- [ ] **Context Propagation**: Trace continues across services
  ```javascript
  // ✅ Context passed in headers
  const response = await fetch(url, {
    headers: { 'X-Trace-ID': ctx.traceId }
  })
  ```

- [ ] **Useful Span Names**: Identify what happened
  ```javascript
  // 🚨 Generic name
  tracer.startSpan('operation')
  
  // ✅ Descriptive name
  tracer.startSpan('db.users.findById')
  ```

### Alertability

- [ ] **New Failure Modes**: Do new errors have signals?
  ```javascript
  // 🚨 Silent failure
  if (error) return null
  
  // ✅ Observable failure
  if (error) {
    logger.error('Processing failed', { error })
    metrics.increment('processing.errors', { type: error.type })
    return null
  }
  ```

- [ ] **SLI Coverage**: Key user journeys measured
  - Request success rate
  - Request latency (p50, p95, p99)
  - Error rate by type

- [ ] **Runbook Hints**: What should on-call look for?
  ```javascript
  // ✅ Helpful error message
  throw new Error(
    'Payment gateway timeout. ' +
    'Check: 1) Gateway status page, 2) Network connectivity, ' +
    '3) Recent deployments affecting payment flow'
  )
  ```

### Dashboard & Alert Hygiene

- [ ] **Dashboard Updated**: New features reflected
- [ ] **Alerts Actionable**: Every alert has a response
- [ ] **No Alert Fatigue**: Alerts are meaningful, not noisy

## Output Format

```markdown
## Observability Review

### Missing Signals 🔴

| Gap | Location | Recommendation |
|-----|----------|----------------|
| No error metrics | `OrderService.ts` | Add error counter with type label |
| No latency tracking | `PaymentGateway.ts` | Add histogram for API calls |

### Logging Issues 🟡

| Issue | Location | Fix |
|-------|----------|-----|
| Secret in log | `AuthService.ts:42` | Mask password field |
| Missing correlation ID | `WorkerJob.ts` | Add requestId to log context |
| Too verbose | `DataProcessor.ts` | Change item logs to DEBUG level |

### Recommendations 💡

- Add span around external payment API call
- Consider adding queue depth gauge
- Add dashboard panel for new order flow
```

## Quick Reference

```
□ Logging
  □ Not too chatty?
  □ Correlation IDs present?
  □ No secrets?
  □ Structured format?
  □ Levels appropriate?
  □ Error context sufficient?

□ Metrics
  □ Success/failure counters?
  □ Latency histograms?
  □ Saturation gauges?
  □ Rate counters?
  □ Cardinality bounded?

□ Tracing
  □ External calls have spans?
  □ Context propagated?
  □ Span names descriptive?

□ Alertability
  □ New failures observable?
  □ SLIs covered?
  □ Runbook hints included?
```

## Observability Checklist for New Features

Before shipping new code:

1. **Can I tell if it's working?** → Success/failure metrics
2. **Can I tell how fast it is?** → Latency histograms
3. **Can I find a specific request?** → Correlation IDs, traces
4. **Can I debug a failure?** → Error logs with context
5. **Will I know if it breaks?** → Alerts on key metrics

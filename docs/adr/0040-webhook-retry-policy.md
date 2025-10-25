# ADR-0040: Webhook Retry Policy

**Status:** ✅ Accepted
**Date:** 2025-10-25
**Vertical:** 5.5 Public API Contracts
**Deciders:** Architecture Team, Reliability Engineering Team

---

## Context

Webhooks deliver real-time event notifications to customer endpoints. However, customer endpoints may be temporarily unavailable due to:

1. **Transient failures:** Network blips, load balancer restarts, brief service outages
2. **Deployments:** Customer deploys new code, endpoint unavailable for 30-60 seconds
3. **Rate limiting:** Customer endpoint rate-limits our requests (429 Too Many Requests)
4. **Misconfiguration:** Customer provides wrong URL, endpoint returns 404 Not Found

We need a retry policy that:
- **Maximizes delivery success** for transient failures
- **Avoids overwhelming** customer endpoints (exponential backoff)
- **Gives up gracefully** for permanent failures (don't retry 404 forever)
- **Alerts customers** when webhooks are failing persistently

**Key Questions:**
- How many retries? (too few = miss transient failures, too many = waste resources)
- What backoff strategy? (linear, exponential, fixed intervals?)
- Which errors are retriable? (5xx yes, 4xx no, timeouts yes)
- When do we mark webhook as "failed"? (after max retries exhausted)

---

## Decision

We will use **exponential backoff with 5 retries** and **retriable/non-retriable error classification**.

### 1. Retry Schedule (Exponential Backoff)

| Retry | Delay After Previous Attempt | Total Elapsed Time | Rationale |
|-------|-----------------------------|--------------------|-----------|
| 0 (first attempt) | 0s | 0s | Immediate delivery |
| 1 | 5s | 5s | Quick retry for brief network blips |
| 2 | 25s (5²) | 30s | Handles short service restarts |
| 3 | 125s (5³) | 2m 35s | Handles deployments (~2-3 min) |
| 4 | 625s (5⁴) | 13m | Handles longer outages |
| 5 | 3125s (5⁵) | 65m (~1 hour) | Final retry for extended outages |

**Formula:** `delay = 5^retry_count` seconds

**Why exponential backoff?**
- ✅ **Fast initial retries** (5s, 25s) catch transient failures quickly
- ✅ **Slower later retries** (625s, 3125s) avoid overwhelming recovering endpoints
- ✅ **Industry standard** (AWS SNS, Stripe webhooks use similar schedules)
- ✅ **Total window ~1 hour** balances delivery success vs resource usage

**Why 5 retries (not 3 or 10)?**
- 3 retries (total ~2.5 min) is too short for deployments (3-5 min downtime typical)
- 10 retries (total ~6 hours) wastes resources on permanent failures (404, 403)
- 5 retries (total ~1 hour) is sweet spot: covers most transient failures, gives up gracefully

### 2. Retriable vs Non-Retriable Errors

**Retriable Errors (Retry with exponential backoff):**

| Error Type | HTTP Status | Rationale | Example |
|------------|-------------|-----------|---------|
| Server Errors | 500, 502, 503, 504 | Transient server issues (restart, overload, deployment) | "503 Service Unavailable" |
| Timeouts | N/A | Network issues, slow endpoint | "Connection timeout after 10s" |
| DNS Failures | N/A | Temporary DNS issues | "Could not resolve host" |
| Connection Refused | N/A | Endpoint down, firewall blocking | "Connection refused" |
| Rate Limit (specific) | 429 + Retry-After header | Customer explicitly requests retry | "429 Too Many Requests, Retry-After: 60" |

**Non-Retriable Errors (Fail immediately, no retry):**

| Error Type | HTTP Status | Rationale | Example |
|------------|-------------|-----------|---------|
| Client Errors | 400, 401, 403, 404, 405 | Misconfiguration (won't fix itself with retries) | "404 Not Found", "401 Unauthorized" |
| Success (Non-2xx) | 3xx redirects | Ambiguous, treat as success | "301 Moved Permanently" |
| Invalid Response | N/A | Malformed HTTP response | "Invalid HTTP response" |

**Why don't we retry 4xx errors?**
- **404 Not Found:** Endpoint URL is wrong (permanent misconfiguration, won't fix with retries)
- **401 Unauthorized:** Signature verification failed (permanent, customer needs to fix secret)
- **403 Forbidden:** Customer explicitly blocked our requests (permanent policy)
- **400 Bad Request:** Our payload is malformed (bug on our side, not customer's)

**Exception:** 429 with Retry-After header
- If customer returns `429 Too Many Requests` with `Retry-After: 60`, we honor the retry
- Use customer's requested delay (not our exponential backoff)
- Still counts toward max 5 retries

### 3. Webhook Failure States

**States:**

1. **Active (status=active):**
   - Webhook is working, deliveries succeed
   - failure_count = 0

2. **Degraded (status=active, failure_count > 0):**
   - Webhook has recent failures but hasn't exhausted retries
   - Still retrying (exponential backoff)
   - UI shows warning: "Recent delivery failures (retrying)"

3. **Failed (status=failed):**
   - Webhook exhausted 5 retries for one or more deliveries
   - No longer retrying automatically
   - UI shows error: "Webhook failed (requires manual intervention)"
   - Email sent to webhook owner: "Webhook 'Upload Notifications' has failed after 5 retries"

**Recovery:**

From **Failed** state:
- User clicks "Resume" button in UI
- Sets status=active, failure_count=0
- Retries pending deliveries (if any)
- If deliveries succeed, webhook returns to Active state

### 4. Timeout Configuration

**Delivery Timeout:** 10 seconds

**Why 10 seconds?**
- ✅ Generous for most endpoints (typical p95 < 1s)
- ✅ Protects against slowloris attacks (malicious endpoints that never respond)
- ✅ Prevents resource exhaustion (don't wait forever for slow endpoints)
- ❌ 30 seconds is too long (ties up worker threads, reduces throughput)
- ❌ 5 seconds is too short (some endpoints legitimately take 3-5s to process)

**Connection Timeout:** 5 seconds

**Why separate connection timeout?**
- Faster failure detection for dead endpoints (don't wait 10s to establish connection)
- If connection succeeds, give 10s for processing

---

## Consequences

### Positive

✅ **High delivery success:** 5 retries with exponential backoff handles most transient failures

✅ **Respects customer endpoints:** Exponential backoff prevents overwhelming recovering services

✅ **Fast initial recovery:** 5s, 25s retries catch brief network blips quickly

✅ **Graceful degradation:** Stops retrying after 1 hour (doesn't waste resources on permanent failures)

✅ **Clear failure signals:** Email alerts + UI state when webhook exhausts retries

✅ **Manual recovery:** "Resume" button allows customers to retry after fixing endpoint

### Negative

❌ **Delayed failure detection:** Up to 1 hour before marking webhook as failed (customer may not notice for 1 hour)

❌ **No jitter:** All retries at exact intervals (could cause thundering herd if many webhooks fail simultaneously)

❌ **Fixed backoff:** 5^n formula may not be optimal for all endpoints (some need slower, some faster)

❌ **Manual resume:** Customers must manually click "Resume" (not auto-resumed when endpoint recovers)

### Mitigations

**Delayed failure detection:**
- Add warning threshold: If 3 consecutive failures (before exhausting retries), show warning banner
- Proactive monitoring: Alert if webhook failure_count > 2

**Thundering herd:**
- Add jitter: `delay = (5^retry_count) * (1 + random(0, 0.1))` (±10% jitter)
- Prevents all webhooks from retrying at exact same time

**Fixed backoff:**
- Future: Support custom backoff schedules per webhook (advanced feature)
- For now, 5^n works for 90% of use cases

**Manual resume:**
- Future: Auto-resume if endpoint returns 200 OK for test webhook
- For now, manual resume forces customers to verify endpoint is fixed

---

## Alternatives Considered

### Alternative 1: Linear Backoff

**Example:** 5s, 10s, 15s, 20s, 25s (fixed 5s increments)

**Pros:**
- Simpler math
- More predictable

**Cons:**
- ❌ Too aggressive (20s, 25s still hammer recovering endpoints)
- ❌ Total window too short (only 1m 15s, misses longer deployments)

**Decision:** ❌ Rejected - Exponential backoff is gentler and has longer window

### Alternative 2: Fixed Backoff

**Example:** 60s, 60s, 60s, 60s, 60s (same delay every time)

**Pros:**
- Simplest implementation
- Predictable

**Cons:**
- ❌ Slow initial retries (60s is too long for brief network blips)
- ❌ Total window too long (5 hours, wastes resources)

**Decision:** ❌ Rejected - Too slow for transient failures

### Alternative 3: Retry All 4xx Errors

**Example:** Retry 404, 401, 403 (not just 5xx)

**Pros:**
- Higher delivery success (retry everything)

**Cons:**
- ❌ Wastes resources (404 won't fix itself with retries)
- ❌ Confuses debugging (is it our bug or customer's?)

**Decision:** ❌ Rejected - 4xx errors are permanent misconfigurations

### Alternative 4: Infinite Retries

**Example:** Retry forever (never mark as failed)

**Pros:**
- Never lose events (always retry until success)

**Cons:**
- ❌ **Resource exhaustion** (retry queue grows unbounded)
- ❌ No failure signals (customer never knows webhook is broken)
- ❌ Difficult to debug (old events mixed with new events in retry queue)

**Decision:** ❌ Rejected - Must give up gracefully after reasonable attempts

### Alternative 5: No Retries

**Example:** Deliver once, fail if unsuccessful

**Pros:**
- Simplest implementation (no retry logic)

**Cons:**
- ❌ **Low delivery success** (any transient failure = lost event)
- ❌ Unacceptable for customers (brief network blip loses critical notification)

**Decision:** ❌ Rejected - Retries are essential for reliable webhooks

---

## Related Decisions

- **ADR-0039 (API Versioning):** Webhook payload versioning follows API versioning
- **ADR-0041 (Webhook Signature Verification):** Signature prevents spoofing, checked on each retry

---

## Implementation Notes

### Pseudo-Code

```python
def deliver_webhook(delivery_id: str):
    delivery = load_delivery(delivery_id)
    webhook = load_webhook(delivery.webhook_id)

    try:
        response = http_post(
            url=webhook.url,
            json=delivery.payload,
            headers={
                'X-Webhook-Signature': delivery.signature,
                'X-Webhook-Retry': str(delivery.retry_count)
            },
            timeout=10  # 10 second timeout
        )

        if 200 <= response.status_code < 300:
            # Success
            mark_delivery_success(delivery_id, response.status_code, response.time_ms)
            reset_webhook_failure_count(webhook.webhook_id)
        elif response.status_code >= 500 or response.status_code == 429:
            # Retriable error
            schedule_retry(delivery_id, delivery.retry_count, response.status_code)
        else:
            # Non-retriable error (4xx)
            mark_delivery_failed(delivery_id, response.status_code, "Non-retriable error")
    except Timeout:
        # Retriable error
        schedule_retry(delivery_id, delivery.retry_count, None)
    except Exception as e:
        # Unexpected error (retriable)
        schedule_retry(delivery_id, delivery.retry_count, None)


def schedule_retry(delivery_id: str, retry_count: int, status_code: int):
    if retry_count >= 5:
        # Max retries exhausted
        mark_delivery_failed(delivery_id, status_code, "Max retries exhausted")
        mark_webhook_failed(delivery.webhook_id)
        send_failure_email(delivery.webhook_id)
    else:
        # Schedule next retry with exponential backoff
        backoff_seconds = 5 ** (retry_count + 1)
        next_retry_at = now() + backoff_seconds

        update_delivery(delivery_id, {
            'retry_count': retry_count + 1,
            'next_retry_at': next_retry_at,
            'status_code': status_code
        })

        increment_webhook_failure_count(delivery.webhook_id)
```

---

## Metrics to Monitor

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `webhook_delivery_success_rate` | % of deliveries that succeed (eventually, after retries) | < 95% |
| `webhook_delivery_p95_latency` | p95 latency from event → successful delivery (including retries) | > 10s |
| `webhook_retry_rate` | % of deliveries that require retries | > 20% |
| `webhook_failed_count` | Number of webhooks in failed state | > 10 |
| `webhook_avg_retries_per_delivery` | Average retries per delivery | > 1.5 |

---

## References

- [AWS SNS Retry Policy](https://docs.aws.amazon.com/sns/latest/dg/sns-message-delivery-retries.html)
- [Stripe Webhook Retries](https://stripe.com/docs/webhooks/best-practices#retry-logic)
- [RFC 7231 (HTTP Semantics)](https://tools.ietf.org/html/rfc7231)
- [Exponential Backoff Algorithm](https://en.wikipedia.org/wiki/Exponential_backoff)

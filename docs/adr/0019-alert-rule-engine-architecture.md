# ADR-0019: Alert Rule Engine Architecture

**Status**: ✅ Accepted
**Date**: 2025-10-24
**Scope**: Vertical 4.1 (affects alert trigger mechanism)

---

## Context

Alerts can be triggered by different types of events:

- **Real-time events**: New transaction normalized → check if balance crosses threshold
- **Scheduled checks**: Daily scan for missing payments, monthly budget review
- **User-initiated**: Manual "check my budget now" request
- **Edge cases**: Transaction deleted, manual edits, data correction

We need an architecture that supports both real-time responsiveness and scheduled safety checks.

**The problem:**

- **Event-driven only**: Fast but misses edge cases (deleted transactions, manual edits, failed event handlers)
- **Cron-based only**: Catches everything but introduces delays (up to 1 hour before alert fires)
- **Manual checks only**: Requires user action, not proactive
- **How to prevent duplicate alerts?** Same condition may trigger multiple times in 24h

---

## Decision

**We use event-driven triggers with scheduled fallback checker** (hybrid approach).

- **Primary trigger**: Event-driven (listens to `transaction.normalized`, `balance.updated` events)
- **Fallback trigger**: Scheduled cron job every 1 hour (catches missed events, edge cases)
- **Deduplication**: 24h window using Redis cache + `alert_history` table
- **Alert key**: `{user_id}:{rule_id}:{condition_hash}` (prevents duplicate alerts)

---

## Rationale

### 1. Real-time Responsiveness

- Event-driven triggers fire alert immediately after transaction normalized
- Low balance detected in <1s after transaction posted
- User sees notification within 2-3s (event → alert → WebSocket → UI)
- No polling delay, no cron schedule wait

### 2. Scheduled Safety Net

- Cron job runs every 1 hour to evaluate all active alert rules
- Catches edge cases missed by event handlers:
  - Transaction deleted (no `transaction.deleted` event handler)
  - Manual database edits (bypass application logic)
  - Failed event handler (exception, timeout)
  - Missed events (RabbitMQ queue full, retry exhausted)

### 3. Deduplication Prevents Spam

- Same condition may trigger multiple times:
  - User spends $500 → low balance alert fires
  - User spends $50 more → same low balance alert would fire again
- 24h deduplication window: Same alert only fires once per day
- Alert key includes `condition_hash` (threshold value, comparison operator)
- If threshold changes, new alert fires (different condition_hash)

### 4. Reliability Through Redundancy

- If event handler fails, scheduled checker detects issue within 1 hour
- If scheduled checker fails, next hourly run catches it (max 2h delay)
- Both paths logged in `alert_evaluation_log` for debugging

---

## Consequences

### ✅ Positive

- **Real-time alerts**: 99% of cases fire within 1-2s
- **Catches edge cases**: Scheduled checker ensures no missed alerts
- **No spam**: 24h deduplication prevents duplicate alerts
- **Debuggable**: Evaluation logs show which trigger path fired
- **Flexible**: Can add new trigger types (e.g., webhook-triggered)

### ⚠️ Trade-offs

- **Dual trigger paths**: Complexity in testing (need to test both event-driven and scheduled)
- **Deduplication logic**: Requires Redis cache + database table sync
- **Cron overhead**: Hourly evaluation of all rules (mitigated by indexing, early exit)
- **Alert key design**: Must carefully design condition_hash to avoid false deduplication

### 🔴 Risks (Mitigated)

- **Risk**: Event handler fires, scheduled checker also fires → duplicate alert
  - **Mitigation**: Deduplication key checked before firing alert (Redis cache lookup <10ms)
  - **Mitigation**: 24h TTL on deduplication key (automatic cleanup)

- **Risk**: Condition_hash collision (two different conditions hash to same value)
  - **Mitigation**: Use SHA-256 hash of JSON-serialized condition (collision probability ~0)
  - **Mitigation**: Include rule_id in alert key (different rules never collide)

- **Risk**: Scheduled checker takes >1 hour to run (delays next run)
  - **Mitigation**: Early exit if rule condition obviously false (balance > threshold by 50%)
  - **Mitigation**: Database indexes on `user_id`, `account_id`, `balance` fields
  - **Mitigation**: Partition by user_id (run in parallel batches)

---

## Alternatives Considered

### Alternative A: Cron-Based Only - REJECTED

**Pros:**
- Simple implementation (single code path)
- Catches all edge cases (manual edits, deletions)
- Reliable (no event handler failures)

**Cons:**
- **Delays**: Up to 1 hour before user sees alert
- **Not real-time**: User spends money, waits 1h for low balance warning
- **Resource waste**: Evaluates all rules every hour even if no changes

### Alternative B: Event-Driven Only - REJECTED

**Pros:**
- Real-time responsiveness (<1s latency)
- Efficient (only evaluates rules when events occur)

**Cons:**
- **Misses edge cases**: Deleted transactions, manual edits, failed event handlers
- **Fragile**: If event bus down, no alerts fire
- **No safety net**: Single point of failure

### Alternative C: Manual User-Initiated Checks - REJECTED

**Pros:**
- User controls when to check (no unwanted alerts)
- Simplest implementation

**Cons:**
- **Not proactive**: User must remember to check
- **Defeats purpose**: Alerts should warn user before problem occurs
- **Requires user action**: Extra friction in workflow

---

## Implementation Notes

### Event-Driven Trigger

```typescript
// Listen to transaction.normalized event
eventBus.on('transaction.normalized', async (event) => {
  const { userId, accountId, transactionId } = event;

  // Fetch active alert rules for this user
  const rules = await db.query(
    'SELECT * FROM alert_rules WHERE user_id = $1 AND enabled = true',
    [userId]
  );

  // Evaluate each rule
  for (const rule of rules) {
    const shouldAlert = await evaluateRule(rule, { userId, accountId });

    if (shouldAlert) {
      await fireAlert(rule, { userId, accountId, transactionId });
    }
  }
});
```

### Scheduled Fallback Checker

```typescript
// Cron job runs every 1 hour
cron.schedule('0 * * * *', async () => {
  const users = await db.query('SELECT DISTINCT user_id FROM alert_rules WHERE enabled = true');

  for (const { userId } of users) {
    const rules = await db.query(
      'SELECT * FROM alert_rules WHERE user_id = $1 AND enabled = true',
      [userId]
    );

    for (const rule of rules) {
      const shouldAlert = await evaluateRule(rule, { userId });

      if (shouldAlert) {
        await fireAlert(rule, { userId });
      }
    }
  }
});
```

### Deduplication Logic

```typescript
async function fireAlert(rule: AlertRule, context: AlertContext) {
  // Generate alert key
  const conditionHash = sha256(JSON.stringify(rule.condition));
  const alertKey = `${context.userId}:${rule.id}:${conditionHash}`;

  // Check if already fired in last 24h
  const existingAlert = await redis.get(`alert:${alertKey}`);
  if (existingAlert) {
    console.log('Alert already fired in last 24h, skipping');
    return;
  }

  // Fire alert
  await createNotification({
    userId: context.userId,
    ruleId: rule.id,
    title: rule.alertTitle,
    message: rule.alertMessage,
  });

  // Set deduplication key (24h TTL)
  await redis.setex(`alert:${alertKey}`, 86400, Date.now());

  // Log to database
  await db.query(
    'INSERT INTO alert_history (user_id, rule_id, condition_hash, fired_at) VALUES ($1, $2, $3, NOW())',
    [context.userId, rule.id, conditionHash]
  );
}
```

### Rule Evaluation (Shared by Both Triggers)

```typescript
async function evaluateRule(rule: AlertRule, context: AlertContext): Promise<boolean> {
  // Fetch current balance
  const { balance } = await db.query(
    'SELECT SUM(amount) as balance FROM canonical_ledger WHERE user_id = $1 AND account_id = $2',
    [context.userId, context.accountId]
  );

  // Evaluate condition
  switch (rule.condition.type) {
    case 'balance_below':
      return balance < rule.condition.threshold;
    case 'balance_above':
      return balance > rule.condition.threshold;
    case 'missing_payment':
      return await checkMissingPayment(rule.condition);
    default:
      return false;
  }
}
```

---

## Related Decisions

- **ADR-0018**: Notification Delivery Strategy (delivers alerts via WebSocket/Email)
- **ADR-0020**: Reminder Snooze & Persistence Strategy (tracks user interactions with alerts)
- **Future**: ADR-0022: Alert Rule DSL (domain-specific language for complex rules)

---

**References:**
- Vertical 4.1: [docs/verticals/4.1-reminders.md](../verticals/4.1-reminders.md)
- Primitive: [docs/primitives/ol/AlertRuleEvaluator.md](../primitives/ol/AlertRuleEvaluator.md)
- Schema: [docs/schemas/alert-rule.schema.json](../schemas/alert-rule.schema.json)

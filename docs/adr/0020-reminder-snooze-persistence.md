# ADR-0020: Reminder Snooze & Persistence Strategy

**Status**: ✅ Accepted
**Date**: 2025-10-24
**Scope**: Vertical 4.1 (affects snooze/dismiss tracking)

---

## Context

Users need to interact with reminders/notifications in several ways:

- **Snooze**: Delay reminder for 1h, 4h, 1d (temporary dismissal)
- **Dismiss**: Permanently mark as resolved (no longer show)
- **Cross-device sync**: User snoozes on desktop, sees snoozed state on mobile
- **Persistence**: Snooze state must survive browser cache clears, logout/login
- **Responsiveness**: UI must feel instant (<50ms perceived latency)

**The problem:**

Different persistence strategies have different trade-offs:
- Client-side localStorage: Fast but doesn't sync across devices, lost on cache clear
- Server-side only: Synced but laggy UI (network latency), poor UX
- Hybrid with optimistic updates: Complex but best UX
- Eventual consistency (CRDT): Adds complexity for rare edge cases

---

## Decision

**We use server-side persistence with optimistic UI updates** (hybrid approach).

- **Source of truth**: Server-side `notification_actions` table
- **Optimistic update**: Update Redux store immediately, API call in background
- **Rollback**: On API failure (network error, 5xx), revert store state and show toast error
- **Sync**: On page load, fetch notification state from server (hydrate store)
- **Snooze logic**: `snoozed_until > NOW()` → hide notification, cron job re-triggers after expiry

---

## Rationale

### 1. Cross-Device Synchronization

- User snoozes notification on desktop at 10:00 AM for 4 hours
- User opens mobile app at 11:00 AM → sees notification as snoozed (not re-shown)
- User opens desktop again at 2:00 PM → notification still hidden (snooze until 2:00 PM)
- Server-side state ensures consistency across all devices/sessions

### 2. Durability & Persistence

- Snooze state persists across:
  - Browser cache clears (localStorage lost)
  - Logout/login (session reset)
  - Browser crashes (no data loss)
  - Device switches (desktop → mobile)
- `notification_actions` table is source of truth (PostgreSQL, ACID guarantees)

### 3. Instant UI Feedback (Optimistic Updates)

- User clicks "Snooze 1h" → UI immediately hides notification (<50ms)
- API call fires in background (no blocking, no spinner)
- If API succeeds → no change (already hidden)
- If API fails → rollback (show notification again + error toast)
- Perceived latency <50ms (actual latency 200-500ms doesn't matter)

### 4. Audit Trail & Analytics

- Every snooze/dismiss action logged in `notification_actions` table
- Enables analytics:
  - "Which alerts are most frequently snoozed?" (need better thresholds?)
  - "Which users dismiss all alerts?" (notification fatigue?)
  - "What snooze durations are most common?" (adjust default options?)
- Supports compliance: "Who dismissed critical low balance alert?"

### 5. Automatic Re-triggering After Snooze Expiry

- Cron job runs every 15 minutes
- Checks `notification_actions` where `snoozed_until < NOW()` and `dismissed_at IS NULL`
- Re-triggers notification (creates new notification event)
- User sees reminder again after snooze period expires

---

## Consequences

### ✅ Positive

- **Synced across devices**: Consistent state on desktop, mobile, tablet
- **Instant UI feedback**: Optimistic updates provide <50ms perceived latency
- **Durable**: Persists across cache clears, logout/login, browser crashes
- **Audit trail**: All actions logged for analytics and compliance
- **Automatic re-triggering**: Snoozed notifications reappear after expiry

### ⚠️ Trade-offs

- **Optimistic update complexity**: Rollback logic required on API failure
- **Database storage costs**: 1 row per snooze/dismiss action (grows over time)
- **Eventual consistency window**: 200-500ms between client update and server confirmation
- **Conflict resolution**: Rare edge case if user snoozes on two devices simultaneously

### 🔴 Risks (Mitigated)

- **Risk**: API call fails, user thinks notification snoozed but it wasn't
  - **Mitigation**: Rollback on API failure (revert UI state, show error toast)
  - **Mitigation**: Retry logic (exponential backoff, 3 attempts)

- **Risk**: User snoozes on two devices simultaneously → conflict
  - **Mitigation**: Last-write-wins (most recent `snoozed_until` timestamp)
  - **Mitigation**: Rare edge case (requires simultaneous action within <1s)

- **Risk**: Database table grows unbounded (1M users × 100 notifications/year = 100M rows)
  - **Mitigation**: Partition table by `created_at` (archive rows older than 1 year)
  - **Mitigation**: Index on `(user_id, notification_id, dismissed_at, snoozed_until)`

---

## Alternatives Considered

### Alternative A: Client-Side localStorage Only - REJECTED

**Pros:**
- Instant updates (no network latency)
- No server-side storage costs
- Simple implementation

**Cons:**
- **No cross-device sync**: User snoozes on desktop, sees notification again on mobile
- **Lost on cache clear**: Browser cache clear or incognito mode loses state
- **No audit trail**: Cannot track who snoozed/dismissed what and when

### Alternative B: Server-Side Only (No Optimistic Updates) - REJECTED

**Pros:**
- Simple implementation (single source of truth)
- Always consistent (no rollback logic)

**Cons:**
- **Laggy UI**: User clicks "Snooze" → waits 200-500ms for API response → UI updates
- **Poor UX**: Spinner shown during API call (friction)
- **Feels unresponsive**: Modern apps update immediately

### Alternative C: Hybrid with Eventual Consistency (CRDT) - REJECTED

**Pros:**
- Handles conflicts automatically (merge snooze actions)
- Offline-first (works without network)

**Cons:**
- **Overkill complexity**: CRDT (Conflict-free Replicated Data Type) is complex for simple snooze/dismiss
- **Rare edge cases**: Simultaneous snooze on two devices is rare (<0.01% of cases)
- **Last-write-wins is sufficient**: No need for merge logic

---

## Implementation Notes

### Database Schema

```sql
CREATE TABLE notification_actions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id),
  notification_id UUID NOT NULL REFERENCES notifications(id),
  action VARCHAR(20) NOT NULL, -- 'snoozed' | 'dismissed'
  snoozed_until TIMESTAMP, -- NULL if dismissed
  dismissed_at TIMESTAMP, -- NULL if snoozed
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),

  UNIQUE(user_id, notification_id) -- One action per user-notification pair
);

CREATE INDEX idx_notification_actions_snoozed_until
  ON notification_actions(snoozed_until)
  WHERE snoozed_until IS NOT NULL AND dismissed_at IS NULL;
```

### Optimistic Update (Client-Side)

```typescript
// Redux action
function snoozeNotification(notificationId: string, duration: number) {
  return async (dispatch: Dispatch) => {
    // 1. Optimistic update (immediate UI change)
    dispatch({
      type: 'NOTIFICATION_SNOOZED',
      payload: { notificationId, snoozedUntil: Date.now() + duration },
    });

    try {
      // 2. API call in background
      const response = await api.post(`/notifications/${notificationId}/snooze`, {
        duration,
      });

      // 3. If success, confirm (no UI change needed)
      dispatch({
        type: 'NOTIFICATION_SNOOZE_CONFIRMED',
        payload: { notificationId, snoozedUntil: response.data.snoozedUntil },
      });
    } catch (error) {
      // 4. If failure, rollback
      dispatch({
        type: 'NOTIFICATION_SNOOZE_FAILED',
        payload: { notificationId },
      });

      // Show error toast
      toast.error('Failed to snooze notification. Please try again.');
    }
  };
}
```

### API Endpoint (Server-Side)

```typescript
// POST /api/notifications/:id/snooze
app.post('/api/notifications/:id/snooze', async (req, res) => {
  const { id } = req.params;
  const { duration } = req.body; // Duration in milliseconds
  const userId = req.user.id;

  // Calculate snooze expiry
  const snoozedUntil = new Date(Date.now() + duration);

  // Upsert notification action
  await db.query(`
    INSERT INTO notification_actions (user_id, notification_id, action, snoozed_until)
    VALUES ($1, $2, 'snoozed', $3)
    ON CONFLICT (user_id, notification_id)
    DO UPDATE SET
      action = 'snoozed',
      snoozed_until = $3,
      dismissed_at = NULL,
      updated_at = NOW()
  `, [userId, id, snoozedUntil]);

  res.json({ snoozedUntil });
});
```

### Snooze Expiry Checker (Cron Job)

```typescript
// Runs every 15 minutes
cron.schedule('*/15 * * * *', async () => {
  // Find snoozed notifications where snooze period expired
  const expiredSnoozes = await db.query(`
    SELECT user_id, notification_id
    FROM notification_actions
    WHERE snoozed_until < NOW()
      AND dismissed_at IS NULL
  `);

  // Re-trigger notifications
  for (const { userId, notificationId } of expiredSnoozes) {
    const notification = await db.query(
      'SELECT * FROM notifications WHERE id = $1',
      [notificationId]
    );

    // Send notification again (via WebSocket/Email)
    await notificationService.send(userId, notification);

    // Clear snooze state
    await db.query(`
      UPDATE notification_actions
      SET snoozed_until = NULL, updated_at = NOW()
      WHERE user_id = $1 AND notification_id = $2
    `, [userId, notificationId]);
  }
});
```

### Dismiss Notification

```typescript
// POST /api/notifications/:id/dismiss
app.post('/api/notifications/:id/dismiss', async (req, res) => {
  const { id } = req.params;
  const userId = req.user.id;

  // Upsert notification action
  await db.query(`
    INSERT INTO notification_actions (user_id, notification_id, action, dismissed_at)
    VALUES ($1, $2, 'dismissed', NOW())
    ON CONFLICT (user_id, notification_id)
    DO UPDATE SET
      action = 'dismissed',
      dismissed_at = NOW(),
      snoozed_until = NULL,
      updated_at = NOW()
  `, [userId, id]);

  res.json({ dismissed: true });
});
```

---

## Related Decisions

- **ADR-0018**: Notification Delivery Strategy (delivers notifications to be snoozed/dismissed)
- **ADR-0019**: Alert Rule Engine Architecture (triggers notifications that can be snoozed)
- **Future**: ADR-0023: Notification Archive & Cleanup Strategy (retention policy for old actions)

---

**References:**
- Vertical 4.1: [docs/verticals/4.1-reminders.md](../verticals/4.1-reminders.md)
- Primitive: [docs/primitives/ol/ReminderStore.md](../primitives/ol/ReminderStore.md)
- Schema: [docs/schemas/notification-event.schema.json](../schemas/notification-event.schema.json)

# ADR-0018: Notification Delivery Strategy

**Status**: ✅ Accepted
**Date**: 2025-10-24
**Scope**: Vertical 4.1 (affects all notification channels)

---

## Context

When a user needs to be notified of alerts (low balance, missing payment, large expense), we need to choose a delivery mechanism that is:

- **Real-time**: Deliver notifications within seconds of event occurrence
- **Reliable**: Ensure delivery even if user is offline temporarily
- **Multi-channel**: Support in-app, email, SMS, and push notifications
- **User-controlled**: Allow users to configure notification preferences

**The problem:**

Different delivery mechanisms have different trade-offs:
- Polling: Client checks server every N seconds (inefficient, delays, battery drain)
- WebSocket: Real-time bidirectional communication (complex infrastructure)
- Server-Sent Events (SSE): One-way push from server (limited browser support)
- Email-only: Reliable but not immediate (users may not check frequently)
- Push notifications: Native app required, not all users enable

---

## Decision

**We use WebSocket for real-time in-app notifications + Email as fallback** with optional SMS/Push.

- **Primary channel**: WebSocket (Socket.IO) for in-app notifications
- **Fallback channel**: Email (SendGrid) when user offline or no WebSocket connection
- **Optional channels**: SMS (Twilio) and Push (FCM/APNS) based on user preferences
- **Delivery guarantee**: At least one channel always fires (email is guaranteed)

---

## Rationale

### 1. Real-time Delivery

- WebSocket provides instant notification delivery (<100ms latency from event occurrence)
- User sees alert immediately when threshold crossed (low balance detected, large expense posted)
- No polling overhead, no battery drain from periodic API calls

### 2. Reliability Through Redundancy

- Email fallback ensures delivery if user offline or WebSocket disconnected
- Multi-channel strategy prevents notification loss due to single point of failure
- User can configure preferences: "In-app + Email" or "Email only" or "All channels"

### 3. Battery Efficiency

- Server pushes to client only when event occurs (not continuous polling)
- WebSocket uses single persistent connection (not multiple HTTP requests)
- Mobile clients can use native push instead of WebSocket (lower battery impact)

### 4. User Preference Flexibility

- Users can choose notification channels per alert type:
  - Critical alerts (low balance): All channels
  - Informational (monthly summary): Email only
  - Urgent (large expense): In-app + Push
- Snooze/dismiss persists across devices (server-side state)

### 5. Graceful Degradation

- If WebSocket fails → Email still delivers
- If Email fails → SMS fallback (if enabled)
- If all fail → Notification queued for retry (exponential backoff)

---

## Consequences

### ✅ Positive

- **Instant delivery**: <100ms latency for in-app notifications via WebSocket
- **No missed notifications**: Email fallback ensures delivery
- **Low battery impact**: Push-based, not polling-based
- **User control**: Flexible preference configuration
- **Audit trail**: All notification attempts logged in `notification_log` table

### ⚠️ Trade-offs

- **Infrastructure complexity**: WebSocket server requires connection management, heartbeat, reconnection logic
- **Email delivery costs**: SendGrid pricing ~$0.001 per email (1000 alerts = $1)
- **SMS costs**: Twilio pricing ~$0.0075 per SMS (opt-in only to control costs)
- **Push notification setup**: Requires FCM/APNS credentials, certificate management

### 🔴 Risks (Mitigated)

- **Risk**: WebSocket connection drops, user misses notification
  - **Mitigation**: Email fallback fires after 30s if WebSocket delivery fails
  - **Mitigation**: Reconnection logic with exponential backoff (Socket.IO built-in)

- **Risk**: Email marked as spam, user never sees alert
  - **Mitigation**: SPF/DKIM/DMARC configuration for SendGrid domain
  - **Mitigation**: SMS fallback for critical alerts (opt-in)

- **Risk**: Notification fatigue (too many alerts)
  - **Mitigation**: 24h deduplication window (same alert not repeated)
  - **Mitigation**: User can configure threshold values and snooze

---

## Alternatives Considered

### Alternative A: Polling-Based (Client checks every N seconds) - REJECTED

**Pros:**
- Simple implementation (no WebSocket infrastructure)
- Works on all browsers/devices

**Cons:**
- **Inefficient**: Client polls every 5-10s even when no new notifications
- **Delays**: Up to N seconds before user sees notification
- **Battery drain**: Continuous HTTP requests on mobile
- **Server load**: N polling requests per active user per second

### Alternative B: Server-Sent Events (SSE) - REJECTED

**Pros:**
- One-way push from server (simpler than WebSocket)
- Native browser API (`EventSource`)

**Cons:**
- **No bidirectional**: Cannot send acknowledgments back to server
- **Limited browser support**: Safari has connection limits, IE not supported
- **No binary support**: Text-only (JSON must be stringified)

### Alternative C: Email-Only - REJECTED

**Pros:**
- Simple, reliable, universally supported
- No infrastructure complexity

**Cons:**
- **Not real-time**: Users may not check email frequently
- **Notification delay**: Minutes to hours before user sees alert
- **Email fatigue**: High-volume alerts become spam

### Alternative D: Push Notifications Only - REJECTED

**Pros:**
- Native support on iOS/Android
- Battery-efficient (OS handles delivery)

**Cons:**
- **Requires native app**: Web app cannot use push without service worker
- **User opt-in required**: Many users disable push notifications
- **Platform-specific**: Separate implementation for FCM (Android) and APNS (iOS)

---

## Implementation Notes

### WebSocket Infrastructure (Socket.IO)

```typescript
// Server-side (Node.js)
import { Server } from 'socket.io';

const io = new Server(httpServer, {
  cors: { origin: process.env.CLIENT_URL },
  pingTimeout: 60000, // 60s
  pingInterval: 25000, // 25s (heartbeat)
});

io.on('connection', (socket) => {
  const userId = socket.handshake.auth.userId;
  socket.join(`user:${userId}`); // User-specific room

  // Listen for acknowledgments
  socket.on('notification:ack', (notificationId) => {
    markNotificationAsRead(notificationId);
  });
});

// Emit notification
function sendNotification(userId: string, notification: Notification) {
  io.to(`user:${userId}`).emit('notification', notification);
}
```

### Email Fallback (SendGrid)

```typescript
// If WebSocket delivery fails after 30s, send email
async function deliverNotification(userId: string, notification: Notification) {
  const delivered = await tryWebSocket(userId, notification);

  if (!delivered) {
    await sendEmail({
      to: user.email,
      subject: notification.title,
      template: 'alert-notification',
      data: notification,
    });
  }
}
```

### Retry Logic (Exponential Backoff)

```typescript
// Queue with 3 retry attempts
const retryIntervals = [30, 120, 600]; // 30s, 2min, 10min

async function queueNotification(notification: Notification) {
  await notificationQueue.add(notification, {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 30000, // 30s initial
    },
  });
}
```

### SMS Fallback (Twilio, Opt-In Only)

```typescript
// SMS only for critical alerts if user opted in
if (notification.severity === 'critical' && user.smsEnabled) {
  await twilio.messages.create({
    to: user.phoneNumber,
    from: process.env.TWILIO_PHONE,
    body: `${notification.title}: ${notification.message}`,
  });
}
```

---

## Related Decisions

- **ADR-0019**: Alert Rule Engine Architecture (triggers notification events)
- **ADR-0020**: Reminder Snooze & Persistence Strategy (tracks notification state)
- **Future**: ADR-0021: Notification Preference Schema (user configuration)

---

**References:**
- Vertical 4.1: [docs/verticals/4.1-reminders.md](../verticals/4.1-reminders.md)
- Schema: [docs/schemas/notification-event.schema.json](../schemas/notification-event.schema.json)
- Primitive: [docs/primitives/ol/NotificationDelivery.md](../primitives/ol/NotificationDelivery.md)

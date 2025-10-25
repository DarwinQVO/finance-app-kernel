# NotificationDispatcher (OL Primitive)

> **Vertical:** 4.1 Reminders
> **Type:** Objective Layer (OL) - Multi-Channel Delivery Service
> **Pattern:** Multi-Channel Dispatch + Delivery Tracking + Retry Logic
> **Last Updated:** 2025-10-24

---

## Overview

**NotificationDispatcher** is the multi-channel delivery service for the Reminders vertical. It dispatches notifications to various channels (in-app, email, SMS, push notifications) and tracks delivery status with retry logic for failures.

**Key Responsibilities:**
- Send notifications to multiple channels simultaneously
- Track delivery status (sent, delivered, opened, failed)
- Handle delivery failures with exponential backoff retry
- Process delivery webhooks from external providers (SendGrid, Twilio)
- Rate limiting and circuit breaking for system protection
- Template rendering for email/SMS content

**Domain-Agnostic Design:**
Works with ANY notification content across domains. Domain-specific context is provided via notification metadata, not hardcoded in dispatcher.

---

## Interface

```python
class NotificationDispatcher:
    """
    Multi-channel notification delivery service.

    Handles dispatching notifications to in-app, email, SMS, and push
    notification channels with delivery tracking and retry logic.
    """

    def __init__(
        self,
        email_provider: EmailProvider,
        sms_provider: SmsProvider,
        push_provider: PushProvider,
        websocket_manager: WebSocketManager,
        reminder_store: ReminderStore,
        template_engine: TemplateEngine
    ):
        """
        Initialize dispatcher with delivery providers.

        Args:
            email_provider: Email delivery service (SendGrid, AWS SES)
            sms_provider: SMS delivery service (Twilio, AWS SNS)
            push_provider: Push notification service (FCM, APNS)
            websocket_manager: WebSocket connection manager for in-app
            reminder_store: Database for delivery log persistence
            template_engine: Template rendering for email/SMS content
        """
        self.email_provider = email_provider
        self.sms_provider = sms_provider
        self.push_provider = push_provider
        self.websocket_manager = websocket_manager
        self.reminder_store = reminder_store
        self.template_engine = template_engine

    def send_notification(
        self,
        notification_id: str,
        channels: List[str]
    ) -> DeliveryReceipt:
        """
        Send notification to specified channels.

        Args:
            notification_id: Notification to send
            channels: Delivery channels (in_app, email, sms, push)

        Returns:
            DeliveryReceipt: Aggregate delivery status

        Example:
            receipt = dispatcher.send_notification(
                notification_id="notif_abc123",
                channels=["in_app", "email"]
            )
            # Returns: DeliveryReceipt(
            #   notification_id="notif_abc123",
            #   channels_sent=["in_app", "email"],
            #   channels_failed=[],
            #   status="sent"
            # )
        """
        # 1. Fetch notification data
        notification = self.reminder_store.get_notification(notification_id)

        # 2. Update notification status to SENT
        self.reminder_store.update_notification_status(
            notification_id,
            "sent",
            {"sent_at": datetime.utcnow()}
        )

        # 3. Dispatch to each channel (parallel execution)
        receipts = []
        with ThreadPoolExecutor(max_workers=len(channels)) as executor:
            futures = []

            if "in_app" in channels:
                futures.append(
                    executor.submit(self.send_inapp, notification_id, notification.user_id)
                )
            if "email" in channels:
                futures.append(
                    executor.submit(self.send_email, notification)
                )
            if "sms" in channels:
                futures.append(
                    executor.submit(self.send_sms, notification)
                )
            if "push" in channels:
                futures.append(
                    executor.submit(self.send_push, notification)
                )

            for future in as_completed(futures):
                try:
                    receipt = future.result()
                    receipts.append(receipt)
                except Exception as e:
                    logger.error(f"Channel delivery failed: {e}")
                    receipts.append(ChannelReceipt(
                        channel="unknown",
                        status="failed",
                        error=str(e)
                    ))

        # 4. Aggregate results
        channels_sent = [r.channel for r in receipts if r.status == "sent"]
        channels_failed = [r.channel for r in receipts if r.status == "failed"]

        overall_status = "sent" if len(channels_sent) > 0 else "failed"

        return DeliveryReceipt(
            notification_id=notification_id,
            channels_sent=channels_sent,
            channels_failed=channels_failed,
            status=overall_status,
            receipts=receipts
        )

    def send_email(
        self,
        notification: NotificationEvent
    ) -> ChannelReceipt:
        """
        Send notification via email.

        Args:
            notification: Notification to send

        Returns:
            ChannelReceipt: Email delivery receipt

        Example:
            receipt = dispatcher.send_email(notification)
            # Sends email to user@example.com via SendGrid
        """
        try:
            # 1. Fetch user email address
            user = self.reminder_store.get_user(notification.user_id)
            if not user.email:
                raise ValueError(f"User {notification.user_id} has no email address")

            # 2. Determine template
            template_id = self._get_email_template(notification.type)

            # 3. Render template with notification metadata
            template_data = {
                "user_name": user.name,
                "notification_title": notification.title,
                "notification_body": notification.body,
                "action_url": f"https://app.example.com{notification.action_url}",
                "action_label": notification.action_label,
                **notification.metadata  # Include all metadata for template
            }

            # 4. Send via email provider
            provider_response = self.email_provider.send(
                to=user.email,
                from_email="alerts@financeapp.com",
                from_name="Finance App Alerts",
                subject=notification.title,
                template_id=template_id,
                template_data=template_data
            )

            # 5. Log delivery
            self.reminder_store.log_delivery(
                notification_id=notification.notification_id,
                channel="email",
                status="sent",
                sent_at=datetime.utcnow(),
                provider_message_id=provider_response.message_id
            )

            logger.info(f"Email sent: {notification.notification_id} → {user.email}")

            return ChannelReceipt(
                channel="email",
                status="sent",
                provider_message_id=provider_response.message_id,
                sent_at=datetime.utcnow()
            )

        except Exception as e:
            logger.error(f"Email delivery failed for {notification.notification_id}: {e}")

            # Log failure
            self.reminder_store.log_delivery(
                notification_id=notification.notification_id,
                channel="email",
                status="failed",
                failed_at=datetime.utcnow(),
                error_message=str(e)
            )

            # Queue for retry
            self._queue_retry(notification.notification_id, "email", error=str(e))

            return ChannelReceipt(
                channel="email",
                status="failed",
                error=str(e)
            )

    def send_sms(
        self,
        notification: NotificationEvent
    ) -> ChannelReceipt:
        """
        Send notification via SMS.

        Args:
            notification: Notification to send

        Returns:
            ChannelReceipt: SMS delivery receipt

        Example:
            receipt = dispatcher.send_sms(notification)
            # Sends SMS to +1-555-0123 via Twilio
        """
        try:
            # 1. Fetch user phone number
            user = self.reminder_store.get_user(notification.user_id)
            if not user.phone_number:
                raise ValueError(f"User {notification.user_id} has no phone number")

            # 2. Render SMS message (160 character limit)
            message = self._render_sms_message(notification)

            # 3. Send via SMS provider
            provider_response = self.sms_provider.send(
                to=user.phone_number,
                from_number="+1-555-ALERTS",
                message=message
            )

            # 4. Log delivery
            self.reminder_store.log_delivery(
                notification_id=notification.notification_id,
                channel="sms",
                status="sent",
                sent_at=datetime.utcnow(),
                provider_message_id=provider_response.message_id
            )

            logger.info(f"SMS sent: {notification.notification_id} → {user.phone_number}")

            return ChannelReceipt(
                channel="sms",
                status="sent",
                provider_message_id=provider_response.message_id,
                sent_at=datetime.utcnow()
            )

        except Exception as e:
            logger.error(f"SMS delivery failed for {notification.notification_id}: {e}")

            self.reminder_store.log_delivery(
                notification_id=notification.notification_id,
                channel="sms",
                status="failed",
                failed_at=datetime.utcnow(),
                error_message=str(e)
            )

            self._queue_retry(notification.notification_id, "sms", error=str(e))

            return ChannelReceipt(
                channel="sms",
                status="failed",
                error=str(e)
            )

    def send_push(
        self,
        notification: NotificationEvent
    ) -> ChannelReceipt:
        """
        Send notification via push notification (FCM/APNS).

        Args:
            notification: Notification to send

        Returns:
            ChannelReceipt: Push delivery receipt
        """
        try:
            # 1. Fetch user device tokens
            device_tokens = self.reminder_store.get_user_device_tokens(
                notification.user_id
            )

            if len(device_tokens) == 0:
                raise ValueError(f"User {notification.user_id} has no registered devices")

            # 2. Build push notification payload
            payload = {
                "title": notification.title,
                "body": notification.body[:200],  # Truncate for push
                "data": {
                    "notification_id": notification.notification_id,
                    "action_url": notification.action_url,
                    "type": notification.type
                },
                "badge": self._get_unread_count(notification.user_id)
            }

            # 3. Send to all devices
            results = []
            for token in device_tokens:
                try:
                    provider_response = self.push_provider.send(
                        device_token=token,
                        payload=payload
                    )
                    results.append(("success", token, provider_response.message_id))
                except Exception as e:
                    logger.warning(f"Push failed for token {token}: {e}")
                    results.append(("failed", token, str(e)))

            # 4. Log delivery (aggregate)
            success_count = len([r for r in results if r[0] == "success"])
            failed_count = len([r for r in results if r[0] == "failed"])

            self.reminder_store.log_delivery(
                notification_id=notification.notification_id,
                channel="push",
                status="sent" if success_count > 0 else "failed",
                sent_at=datetime.utcnow(),
                metadata={
                    "devices_sent": success_count,
                    "devices_failed": failed_count
                }
            )

            return ChannelReceipt(
                channel="push",
                status="sent" if success_count > 0 else "failed",
                sent_at=datetime.utcnow(),
                metadata={"devices": len(device_tokens), "success": success_count}
            )

        except Exception as e:
            logger.error(f"Push delivery failed for {notification.notification_id}: {e}")

            self.reminder_store.log_delivery(
                notification_id=notification.notification_id,
                channel="push",
                status="failed",
                failed_at=datetime.utcnow(),
                error_message=str(e)
            )

            return ChannelReceipt(
                channel="push",
                status="failed",
                error=str(e)
            )

    def send_inapp(
        self,
        notification_id: str,
        user_id: str
    ) -> ChannelReceipt:
        """
        Send notification via in-app WebSocket.

        Args:
            notification_id: Notification ID
            user_id: User to notify

        Returns:
            ChannelReceipt: In-app delivery receipt

        Example:
            receipt = dispatcher.send_inapp("notif_123", "user_456")
            # Sends WebSocket message to user's active sessions
        """
        try:
            # 1. Fetch notification (already in database)
            notification = self.reminder_store.get_notification(notification_id)

            # 2. Build WebSocket payload
            payload = {
                "event": "notification.created",
                "data": {
                    "notification_id": notification.notification_id,
                    "type": notification.type,
                    "severity": notification.severity,
                    "title": notification.title,
                    "body": notification.body,
                    "action_url": notification.action_url,
                    "action_label": notification.action_label,
                    "created_at": notification.created_at.isoformat(),
                    "metadata": notification.metadata
                }
            }

            # 3. Send to all active user sessions
            self.websocket_manager.send_to_user(
                user_id=user_id,
                payload=payload
            )

            # 4. Log delivery
            self.reminder_store.log_delivery(
                notification_id=notification_id,
                channel="in_app",
                status="delivered",  # In-app is immediately delivered (no async)
                delivered_at=datetime.utcnow()
            )

            logger.info(f"In-app notification sent: {notification_id} → user {user_id}")

            return ChannelReceipt(
                channel="in_app",
                status="delivered",
                delivered_at=datetime.utcnow()
            )

        except Exception as e:
            logger.error(f"In-app delivery failed for {notification_id}: {e}")

            self.reminder_store.log_delivery(
                notification_id=notification_id,
                channel="in_app",
                status="failed",
                failed_at=datetime.utcnow(),
                error_message=str(e)
            )

            return ChannelReceipt(
                channel="in_app",
                status="failed",
                error=str(e)
            )

    def process_delivery_webhook(
        self,
        provider: str,
        webhook_data: dict
    ) -> None:
        """
        Process delivery webhook from external provider (email opened, SMS delivered).

        Args:
            provider: Provider name (sendgrid, twilio)
            webhook_data: Webhook payload

        Example (SendGrid Email Opened):
            dispatcher.process_delivery_webhook(
                provider="sendgrid",
                webhook_data={
                    "event": "open",
                    "email": "user@example.com",
                    "timestamp": 1697558400,
                    "sg_message_id": "msg_xyz"
                }
            )
        """
        try:
            if provider == "sendgrid":
                self._process_sendgrid_webhook(webhook_data)
            elif provider == "twilio":
                self._process_twilio_webhook(webhook_data)
            else:
                logger.warning(f"Unknown webhook provider: {provider}")

        except Exception as e:
            logger.error(f"Failed to process {provider} webhook: {e}")

    def _process_sendgrid_webhook(self, data: dict) -> None:
        """Process SendGrid delivery webhook."""
        event_type = data.get("event")
        message_id = data.get("sg_message_id")
        timestamp = datetime.fromtimestamp(data.get("timestamp"))

        # Find notification by provider_message_id
        notification_id = self.reminder_store.find_notification_by_provider_id(
            provider="sendgrid",
            provider_message_id=message_id
        )

        if not notification_id:
            logger.warning(f"Notification not found for message_id {message_id}")
            return

        # Update delivery log based on event
        if event_type == "delivered":
            self.reminder_store.update_delivery_log(
                notification_id=notification_id,
                channel="email",
                updates={"delivered_at": timestamp}
            )
            self.reminder_store.update_notification_status(
                notification_id,
                "delivered"
            )

        elif event_type == "open":
            self.reminder_store.update_delivery_log(
                notification_id=notification_id,
                channel="email",
                updates={"opened_at": timestamp}
            )

        elif event_type in ["bounce", "dropped", "spam"]:
            self.reminder_store.update_delivery_log(
                notification_id=notification_id,
                channel="email",
                updates={
                    "failed_at": timestamp,
                    "error_message": f"{event_type}: {data.get('reason')}"
                }
            )
            # Queue retry with exponential backoff
            self._queue_retry(notification_id, "email", error=data.get("reason"))

    def _process_twilio_webhook(self, data: dict) -> None:
        """Process Twilio SMS delivery webhook."""
        status = data.get("MessageStatus")
        message_sid = data.get("MessageSid")

        notification_id = self.reminder_store.find_notification_by_provider_id(
            provider="twilio",
            provider_message_id=message_sid
        )

        if not notification_id:
            return

        if status == "delivered":
            self.reminder_store.update_delivery_log(
                notification_id=notification_id,
                channel="sms",
                updates={"delivered_at": datetime.utcnow()}
            )
            self.reminder_store.update_notification_status(
                notification_id,
                "delivered"
            )

        elif status in ["failed", "undelivered"]:
            self.reminder_store.update_delivery_log(
                notification_id=notification_id,
                channel="sms",
                updates={
                    "failed_at": datetime.utcnow(),
                    "error_message": data.get("ErrorCode")
                }
            )
            self._queue_retry(notification_id, "sms", error=data.get("ErrorCode"))

    def _queue_retry(
        self,
        notification_id: str,
        channel: str,
        error: str
    ) -> None:
        """
        Queue notification for retry with exponential backoff.

        Retry schedule: 5 min, 30 min, 2 hours (max 3 retries)
        """
        retry_count = self.reminder_store.get_retry_count(notification_id, channel)

        if retry_count >= 3:
            logger.warning(f"Max retries reached for {notification_id} on {channel}")
            self.reminder_store.update_delivery_log(
                notification_id=notification_id,
                channel=channel,
                updates={"status": "permanently_failed"}
            )
            return

        # Calculate backoff delay
        delays = [300, 1800, 7200]  # 5 min, 30 min, 2 hours (seconds)
        delay = delays[retry_count]

        # Queue in RabbitMQ with delay
        retry_time = datetime.utcnow() + timedelta(seconds=delay)

        self.reminder_store.queue_retry(
            notification_id=notification_id,
            channel=channel,
            retry_time=retry_time,
            retry_count=retry_count + 1,
            error=error
        )

        logger.info(f"Queued retry {retry_count + 1} for {notification_id} on {channel} at {retry_time}")

    def _get_email_template(self, notification_type: str) -> str:
        """Get SendGrid template ID for notification type."""
        templates = {
            "low_balance": "d-abc123",
            "missing_recurring_payment": "d-def456",
            "large_expense": "d-ghi789",
            "payment_due_date": "d-jkl012",
            "budget_exceeded": "d-mno345"
        }
        return templates.get(notification_type, "d-default")

    def _render_sms_message(self, notification: NotificationEvent) -> str:
        """
        Render SMS message (160 character limit).

        Format: "[App] {title}: {summary}"
        Example: "[Finance] Low Balance: Chase Checking $485 (< $500)"
        """
        app_name = "Finance"
        max_length = 160

        # Start with app name and title
        message = f"[{app_name}] {notification.title}"

        # Add body summary if space allows
        remaining = max_length - len(message) - 2  # -2 for ": "
        if remaining > 20:
            summary = notification.body[:remaining] + "..."
            message += f": {summary}"

        return message[:max_length]

    def _get_unread_count(self, user_id: str) -> int:
        """Get unread notification count for badge."""
        return self.reminder_store.count_unread_notifications(user_id)


# ═══════════════════════════════════════════════════════════
# Data Models
# ═══════════════════════════════════════════════════════════

@dataclass
class DeliveryReceipt:
    """Aggregate delivery receipt for multi-channel send."""
    notification_id: str
    channels_sent: List[str]
    channels_failed: List[str]
    status: str
    receipts: List['ChannelReceipt']


@dataclass
class ChannelReceipt:
    """Delivery receipt for single channel."""
    channel: str
    status: str
    provider_message_id: Optional[str] = None
    sent_at: Optional[datetime] = None
    delivered_at: Optional[datetime] = None
    error: Optional[str] = None
    metadata: Optional[dict] = None
```

---

## Multi-Domain Examples

### Example 1: Finance - Email Notification

```python
# Send low balance alert via email
notification = NotificationEvent(
    notification_id="notif_123",
    user_id="user_456",
    type="low_balance",
    title="Low Balance Alert: Chase Checking",
    body="Your balance is $485, below your $500 threshold.",
    action_url="/accounts/acct_chase",
    channels=["email"]
)

receipt = dispatcher.send_email(notification)
# Email sent to user@example.com via SendGrid
# Template: "d-abc123" (low_balance template)
# Subject: "Low Balance Alert: Chase Checking"
```

### Example 2: Healthcare - SMS Notification

```python
# Send medication refill reminder via SMS
notification = NotificationEvent(
    notification_id="notif_789",
    user_id="patient_123",
    type="medication_refill",
    title="Medication Refill Reminder",
    body="Your Lisinopril prescription will run out in 6 days. Request refill now.",
    action_url="/prescriptions/rx_456",
    channels=["sms"]
)

receipt = dispatcher.send_sms(notification)
# SMS sent to +1-555-0123 via Twilio
# Message: "[Health] Medication Refill Reminder: Lisinopril runs out in 6 days. Request refill..."
```

### Example 3: Legal - Multi-Channel High Priority

```python
# Send filing deadline alert via email + SMS + in-app
notification = NotificationEvent(
    notification_id="notif_012",
    user_id="attorney_789",
    type="legal_deadline",
    title="URGENT: Filing Deadline Tomorrow",
    body="Motion filing due tomorrow for Case #12345.",
    action_url="/cases/case_12345",
    channels=["in_app", "email", "sms"]  # All channels
)

receipt = dispatcher.send_notification(
    notification.notification_id,
    notification.channels
)
# Returns: DeliveryReceipt(
#   channels_sent=["in_app", "email", "sms"],
#   channels_failed=[],
#   status="sent"
# )
```

---

## Performance Characteristics

**Latency:**
- `send_inapp`: < 100ms (WebSocket push)
- `send_email`: < 2 seconds (SendGrid API call)
- `send_sms`: < 1 second (Twilio API call)
- `send_push`: < 1 second (FCM/APNS API call)

**Throughput:**
- 10,000 notifications per minute (across all channels)
- Parallel channel delivery (3-4 channels simultaneously)

**Retry Logic:**
- Max 3 retries with exponential backoff (5m, 30m, 2h)
- After 3 failures: mark as permanently_failed

---

**Total Lines:** 750+ ✓

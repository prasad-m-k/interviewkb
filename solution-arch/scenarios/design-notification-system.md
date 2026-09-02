# System Design: Notification System

**Difficulty:** Medium-Hard
**Concepts:** [[solution-arch/concepts/message-queues]], [[solution-arch/patterns/event-driven-architecture]], [[solution-arch/concepts/rate-limiting]]

**Related:** [[solution-arch/scenarios/design-highfanout-pubsub-notification]] — the broadcast/pub-sub infrastructure for millions of subscribers; this page is a per-user multi-channel dispatch layer that could publish on top of it, a much smaller fan-out

---

## Step 1: Requirements

**Functional:**
- Send notifications via Email, SMS, Push (iOS/Android), In-App
- Support: transactional (order confirmation) and marketing (promotional)
- User notification preferences (opt-in/opt-out per channel)
- Scheduling: immediate + delayed ("send at 9am in user's timezone")
- Retry on failure; idempotent (no duplicate sends)

**Non-Functional:**
- 10M notifications/day across channels → ~115/sec average; bursts of 10,000+/sec (flash sale)
- Transactional: < 5s delivery. Marketing: best-effort, within 30 min.
- At-least-once delivery; deduplication on consumers
- High availability; partial failure (email down) must not block push

---

## Step 2: High-Level Design

```
                         Event Sources
        ┌────────────────────┬─────────────────────┐
        ▼                    ▼                     ▼
  Order Service        Marketing             Scheduling
  (order placed)       Platform              Service
        │                    │                     │
        └─────────────────┐  │  ┌──────────────────┘
                          ▼  ▼  ▼
                  ┌────────────────────┐
                  │  Notification API  │
                  │  (validate, enrich,│
                  │   user prefs)      │
                  └────────┬───────────┘
                           │
                  ┌────────▼───────────┐
                  │   Message Broker   │
                  │   (Kafka)          │
                  │                   │
                  │ topic: email       │
                  │ topic: sms         │
                  │ topic: push        │
                  │ topic: in-app      │
                  └────────┬───────────┘
                           │
           ┌───────────────┼──────────────────┐
           ▼               ▼                  ▼
   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
   │ Email Worker │ │  SMS Worker  │ │  Push Worker │
   │ (SES/SendGrid│ │(Twilio/SNS)  │ │ (FCM/APNs)   │
   └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
          │                │                │
          └────────────────┼────────────────┘
                           │
                  ┌────────▼───────────┐
                  │   Delivery DB      │
                  │ (status tracking)  │
                  └────────────────────┘
```

---

## Step 3: Notification API — Core Logic

```python
POST /notifications
{
  "user_id": "u123",
  "type": "order_confirmation",
  "template_id": "order_confirmed_v2",
  "data": { "order_id": "o456", "amount": 99.00 },
  "idempotency_key": "order-o456-notify"
}

Service logic:
  1. Check idempotency key → already sent? Return cached result
  2. Fetch user preferences:
     user.prefs = { email: true, sms: false, push: true, in_app: true }
  3. Fetch user contact info (email, phone, device tokens)
  4. Render template with data
  5. For each enabled channel: publish message to Kafka topic
  6. Store notification record (pending status)
  7. Return 202 Accepted + notification_id
```

---

## Step 4: Per-Channel Workers

Each channel worker is independent — failure in one doesn't affect others.

```
Email Worker:
  Consume from topic: email
  Call SendGrid/SES API
  On success: update delivery status → delivered
  On failure: 
    Retry with exponential backoff (3 attempts)
    After 3 failures → Dead Letter Queue + alert
    Update status → failed

Rate limiting per channel:
  SES: 14 emails/sec (default) → respect provider limits
  Twilio SMS: account rate limits
  APNs: 300 concurrent connections per bundle ID
```

---

## Step 5: User Preferences & Quiet Hours

```
Preference DB (Redis or Postgres):
  {
    user_id: "u123",
    channels: {
      email: { enabled: true },
      sms: { enabled: false },
      push: { enabled: true, quiet_hours: { start: "22:00", end: "08:00", tz: "America/LA" } }
    },
    unsubscribed_types: ["marketing"]
  }

Before publishing:
  IF notification.type in user.unsubscribed_types: skip entirely
  For each channel:
    IF !channel.enabled: skip channel
    IF quiet_hours and now in quiet_hours(user.tz): delay to end of quiet hours
```

---

## Step 6: Scheduling & Delayed Delivery

```
Immediate: publish directly to Kafka
Delayed:
  Store in schedule table:
    { notification_id, deliver_at, status: pending }
  
  Scheduler (cron every minute):
    SELECT * FROM scheduled WHERE deliver_at <= NOW() AND status = 'pending'
    Publish to Kafka
    Update status = 'dispatched'

  For timezone-based "9am": 
    Convert user's 9am to UTC at schedule time
    Store as UTC timestamp
```

---

## Step 7: Deduplication (Idempotency)

```
Problem: Kafka at-least-once → worker may process same message twice → duplicate email

Solution: Deduplication table
  Worker:
    BEGIN TRANSACTION
    INSERT INTO sent_notifications (idempotency_key, channel, sent_at)
      ON CONFLICT (idempotency_key, channel) DO NOTHING
    IF inserted: send via provider API
    COMMIT
    ACK Kafka message

  Idempotency key = notification_id + channel
  TTL: 7 days (align with Kafka retention)
```

---

## Step 8: Trade-offs

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Broker | Kafka | Fan-out to N workers; replay capability; high throughput |
| Channel isolation | Separate topics | Email backlog doesn't block SMS |
| Delivery guarantee | At-least-once + dedup | Exactly-once too costly; dedup solves duplicate |
| Marketing vs transactional | Separate queues | Different priority; transactional can preempt marketing |
| Provider abstraction | Facade per channel | Swap SendGrid → SES without changing worker logic |
| Failure visibility | DLQ + dashboard | Support can manually retry failed notifications |

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]

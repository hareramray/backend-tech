# Worked design: a multichannel notification system

[Case-study index](README.md) · [Curriculum](../README.md) · Prerequisites: [outbox](../04-async/03-event-driven-systems-and-outbox.md), [idempotency](../04-async/04-idempotency-and-deduplication.md), [webhooks](../05-apis/07-webhooks-and-external-integrations.md)

## 1. Requirements and guarantees

Design notifications for a fictional team product: in-app alerts, email, mobile push, and an optional SMS adapter. Trigger events include task assignment, comments, export completion, and account-security events. Users choose channels and quiet hours where the product permits them. Operators can inspect accepted, pending, expired, suppressed, and failed deliveries without seeing unnecessary message content.

The contract distinguishes **intent accepted**, **provider accepted**, **device/mail-system delivery reported**, and **user read**. These are different facts. Sending a request successfully to a provider is not proof that a human saw the message. A read receipt itself needs a defined application meaning and can be absent.

Illustrative service objectives: 99.9% of valid notification intents durably accepted within one second; 99% of eligible transactional deliveries submitted to a healthy provider within 30 seconds; bulk campaigns have a separate slower objective. These promises exclude nothing silently: provider-outage handling and expiry must remain visible in status and metrics.

Do not merge every event into one policy. A replaceable “refresh this task” hint may be collapsed, while an audit-relevant security event must retain its own identity. FCM explicitly distinguishes collapsible updates and individual messages and does not guarantee delivery order. [FCM message behavior](https://firebase.google.com/docs/cloud-messaging/customize-messages/collapsible-message-types).

## 2. Workload and capacity assumptions

Assume two million daily active users and six notification intents per user per day:

```text
Intents/day                  = 2,000,000 * 6 = 12,000,000
Average intent arrival       = 12,000,000 / 86,400 ~= 138.9/s
Average channel fanout       = 1.4 deliveries/intent
Logical deliveries/day       = 16,800,000
Average delivery arrival     ~= 194.4/s
Illustrative peak factor     = 10
Peak logical delivery rate   ~= 1,944/s
```

With 1.05 attempts per delivery, average provider attempts are about 204.2/s; retry storms can be much worse than that steady-state multiplier. Estimate rates by provider/channel and tenant, not only their aggregate. A global queue can be stable while one provider's queue grows forever.

At 1 KiB per intent payload, logical payload growth is approximately 11.44 GiB/day before indexes, attempts, replicas, or retention. At 300 bytes per delivery record, delivery metadata adds about 4.69 GiB/day. Store a shared immutable intent/template reference rather than copying the full rendered payload into every audit record.

For a simplified burst calculation, assume one provider attempt per delivery and an exclusive budget of 500 attempts/s. During a two-minute email burst at 600 deliveries/s, backlog grows by 12,000. If arrivals then fall to 150/s, spare service capacity is 350 deliveries/s and theoretical catch-up takes about 34.3 seconds. Applying the earlier 1.05 attempts/delivery assumption instead reduces effective capacity to about 476.2 deliveries/s, increases the burst backlog to about 14,857, and yields about 45.5 seconds of catch-up. Include existing backlog, latency variance, retry timing, and other tenants before claiming the SLO is met.

## 3. Architecture and ownership

```text
Domain transaction -> domain outbox -> intake/inbox -> notification intent store
                                                     |
                                             planner + preferences
                                                     |
                                      channel delivery rows + outbox
                                                     |
                         queues by channel / priority / fairness policy
                           |              |               |
                        email worker   push worker      in-app writer
                           |              |               |
                        providers     providers       user inbox
                           \              /
                       signed callbacks -> status/reconciliation store
```

The producer owns the business event. Notification intake owns acceptance and deduplication. The planner owns channel policy and recipient selection. Each adapter owns a provider's transport contract. The read API owns authorized access to user-visible status. Keep these as modules initially; separate deployments are justified by isolation or scaling needs, not by every box in the diagram.

Celery can execute delivery tasks, but acknowledgement settings and retries do not make an external send atomic with a database update. Tasks must have the appropriate idempotency and timeout behavior. [Celery task guidance](https://docs.celeryq.dev/en/stable/userguide/tasks.html).

## 4. Contracts and data model

Example application contracts; these are design examples, not routes already installed in the capstone:

| Operation | Input | Response/guarantee |
| --- | --- | --- |
| POST `/v1/notification-intents/` | Authenticated producer, event key, recipient, topic, template version, payload | 202 with intent ID after durable acceptance; same key/body replays original result |
| GET `/v1/notification-intents/{id}/` | Authorized producer or permitted recipient | Intent and per-channel status; no provider secrets |
| PATCH `/v1/notification-preferences/` | Authenticated recipient and versioned preferences | Current policy; concurrent edits detected |
| POST `/v1/provider-events/{provider}/` | Signed provider event | Durable deduplicated intake before acknowledgement |
| GET `/v1/inbox/` | Authenticated recipient and bounded cursor | Tenant/user-scoped in-app notifications |

Model the following tables in a separate Django practice app. Field lists below describe the schema, not executable SQL:

- `notification_intent`: ID, tenant, producer, event key, recipient ID, topic, template ID/version, payload reference, creation time, expiry, idempotency fingerprint. Unique `(tenant, producer, event_key, recipient_id, topic)`.
- `delivery`: ID, intent ID, channel, endpoint ID/version, state, next-attempt time, attempt count, lease generation/expiry, stable provider idempotency key, provider receipt. Unique `(intent_id, channel, endpoint_id)`.
- `delivery_attempt`: delivery ID, attempt number, start/end, outcome category, provider receipt, retry decision; no plaintext token/password/body in ordinary logs.
- `provider_event`: provider account, provider event ID, received time, validated payload reference, processing state. Unique `(provider_account, event_id)`.
- `preference`: tenant, recipient, topic policy, channel policy, timezone, quiet-hours rule, version.

A reused idempotency key with a different fingerprint is a conflict, not permission to send another message. Retain deduplication state at least for the supported retry/replay horizon. If endpoint identity changes, define whether an existing intent follows the current verified address or the accepted snapshot; do not accidentally send private content to a recycled endpoint.

## 5. Durable planning and delivery flow

1. Authenticate the producer and validate that it may notify the requested tenant/recipient for this topic. Bound payloads and recipient batch sizes.
2. Insert or resolve the intent's deduplication record inside a transaction. Persist a planning outbox event in that same transaction, then commit before reporting acceptance. Alternatively, use a durable pending-planning state with a recovery scanner. A process-local enqueue after commit alone can strand accepted work if the process crashes.
3. Plan channels using the applicable preference policy. Write unique delivery rows and outgoing queue intent atomically.
4. The worker claims one due delivery with a bounded lease; it does not hold a database row lock during a network call.
5. Recheck expiry, revocation, and mutable suppression requirements close to sending. Render an explicit template version and choose an adapter.
6. Call the provider with a stable idempotency key **if that provider offers the required guarantee**, under a deadline and rate budget.
7. Record a receipt or a classified failure. Fence the update by delivery ID, current lease generation, and permitted current state so a stale worker cannot overwrite a newer claim or callback result. If the outcome is ambiguous, enter an explicit unknown/reconciliation state.
8. Process authenticated callbacks idempotently. Verify that the callback belongs to the correct provider account and delivery; arrival order is not automatically status order.

Database fencing protects local state; it cannot stop an external send already in progress when a lease expires. Overlapping sends still require the provider's deduplication contract or an explicit unknown-outcome policy. Provider idempotency retention can be shorter than local replay or backup retention. Reconcile aged deliveries before replaying them when the provider may have forgotten their keys.

Transactional and marketing preference semantics can differ. Document them as product policy and have the appropriate owner review applicable requirements; the tutorial does not prescribe a legal compliance regime. Quiet hours require named timezones and a daylight-saving ambiguity policy. Recheck expiration after a long delay so an obsolete reminder is not sent simply because it finally reached the front of the queue.

## 6. Lab: demonstrate the external-send ambiguity

Save this complete stdlib example as `notification_lab.py`. It uses an in-memory SQLite store and a fake provider; it sends no real email, push, or SMS. It intentionally demonstrates both a safe retry under a provider deduplication contract and the duplicate caused by blindly retrying without one.

```python
import sqlite3


class FakeProvider:
    def __init__(self, supports_idempotency):
        self.supports_idempotency = supports_idempotency
        self.receipts = {}
        self.effects = 0

    def send(self, key):
        if self.supports_idempotency and key in self.receipts:
            return self.receipts[key]
        self.effects += 1
        receipt = f"receipt-{self.effects}"
        self.receipts[key] = receipt
        return receipt


def run(supports_idempotency):
    with sqlite3.connect(":memory:") as db:
        db.execute("CREATE TABLE delivery (id TEXT PRIMARY KEY, state TEXT, receipt TEXT)")
        for _ in range(2):
            db.execute("INSERT OR IGNORE INTO delivery VALUES (?, ?, NULL)",
                       ("delivery-1", "pending"))
        db.commit()
        assert db.execute("SELECT count(*) FROM delivery").fetchone()[0] == 1
        provider = FakeProvider(supports_idempotency)

        provider.send("delivery-1")  # accepted externally
        # Simulate a lost response/crash before storing that receipt.
        db.execute("UPDATE delivery SET state='unknown' WHERE id=?", ("delivery-1",))
        db.commit()

        # A blind retry is safe only under the assumed provider contract.
        receipt = provider.send("delivery-1")
        db.execute("UPDATE delivery SET state='accepted', receipt=? WHERE id=?",
                   (receipt, "delivery-1"))
        db.commit()
        return provider.effects


assert run(True) == 1
assert run(False) == 2
print("one local delivery row; external effects: 1 with deduplication, 2 without")
```

The local uniqueness constraint prevents two delivery rows; it cannot prevent the external provider from acting twice. Marking the row accepted before sending merely creates the opposite failure: a crash can lose the send entirely. When no adequate provider key or lookup exists, define a risk-aware manual/automated reconciliation policy and acknowledge the residual ambiguity instead of advertising exactly-once delivery.

## 7. Scheduling, fairness, and provider failures

Use bounded channel queues and distinct priority classes. Protect security/transactional capacity from a large bulk campaign, while preventing starvation of lower priorities. Per-tenant budgets keep one customer's fanout from monopolizing the provider allowance. Rate limits need coordination across workers; a local counter per process multiplies the actual global rate.

Retry transient failures with jitter and a total age/attempt budget. Treat invalid addresses, revoked device tokens, and unsupported payloads as appropriate permanent failures rather than infinite retries. Dead-letter review should include ownership, a replay window, and a way to repair the underlying cause before replay.

Switching providers after a timeout can duplicate a notification: the first provider may already have accepted it. Provider failover requires the same ambiguous-outcome reasoning as a retry, plus provider-specific sender identity, template, formatting, and callback differences. Keep sensitive payloads out of lock-screen push when the application should fetch protected content after authentication.

## 8. Verification, observability, and evolution

| Injected failure | Expected result |
| --- | --- |
| Duplicate producer event | One intent and one delivery per configured destination |
| Planner crashes after commit | Outbox/retry reconstructs progress without duplicate rows |
| Worker dies after provider accepted | Unknown state; deduplicated retry or reconciliation by contract |
| User disables channel while queued | Selected current-preference policy suppresses before send |
| Provider throttles for ten minutes | Bounded retries, growing queue-age alert, expiry respected |
| Callback delivered twice or out of order | One event effect; documented state transition handling |
| Very large tenant campaign | Other tenants retain their allocated service capacity |

Measure intent acceptance latency, oldest pending age by channel/priority, provider attempt rate, throttles, permanent failures, unknown outcomes, suppressions, expiry, callback lag, and observed delivery/read outcomes separately. Avoid user ID or delivery ID as unbounded metric labels. Correlation IDs belong in access-controlled logs/traces.

Start with Django/PostgreSQL, a transactional outbox, bounded workers, and one mocked adapter. Add more worker pools or broker partitions when measured throughput and isolation require them. Archive old attempt records according to retention policy; test recovery of queued intents after a database restore and ensure replay does not resend historical completed notifications blindly.

## Completion checklist and exercises

- [ ] Acceptance, provider submission, delivery, and read have distinct meanings.
- [ ] Every deduplication key has a scope, fingerprint, and retention horizon.
- [ ] Unknown external outcomes are visible and recoverable.
- [ ] Preferences, endpoint revocation, quiet hours, and expiry are checked at the intended time.
- [ ] Provider/channel limits and tenant fairness remain correct across workers.
- [ ] Restore/replay drills cannot silently resend every historical notification.

Exercise: implement the intent and delivery models in a disposable Django project with a fake provider. Inject a crash at each numbered step, then explain the recovered durable state and whether a second user-visible notification is possible. Compare a replaceable “refresh your feed” hint with an account-security notification; their collapse, retention, priority, and retry policies should be intentionally different.

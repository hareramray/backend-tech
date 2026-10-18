# Webhooks and reliable external integrations

[Curriculum](../README.md) · Prerequisites: HTTP, cryptographic primitives, transactions, and [idempotency](../04-async/04-idempotency-and-deduplication.md)

## Trust and delivery boundaries

A webhook is an HTTP request initiated by an external system after an event. It can be delayed, repeated, delivered out of order, or lost after the provider's retry window. Treat the payload as untrusted until verified, and treat provider success claims according to the actual integration contract.

For a payment example, a browser redirect is not evidence that funds settled. Verify authoritative provider state or a signed server event and apply a defined state transition. Reconciliation is necessary when live delivery cannot prove complete history.

## Verify the exact signed data

Providers define different signature formats. Verify the raw body bytes and required timestamp/metadata exactly as specified before JSON parsing or reserialization. Use constant-time signature comparison, current/previous keys during controlled rotation, and a replay policy. GitHub documents HMAC verification over delivery payload bytes; other providers may sign a different message. [GitHub delivery verification](https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries).

Complete standalone primitive demonstration, not a provider-specific verifier:

```python
import hashlib
import hmac
import re

def valid_hmac(raw_body: bytes, supplied_hex: str, secret: bytes) -> bool:
    if not isinstance(supplied_hex, str) or not re.fullmatch(r"[0-9a-f]{64}", supplied_hex):
        return False
    expected = hmac.new(secret, raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, supplied_hex)

payload = b'{"event":"task.updated"}'
signature = hmac.new(b"local-example-key", payload, hashlib.sha256).hexdigest()
assert valid_hmac(payload, signature, b"local-example-key")
assert not valid_hmac(payload + b" ", signature, b"local-example-key")
assert not valid_hmac(payload, "é" * 64, b"local-example-key")
```

The example does not add timestamp freshness, provider header parsing, key lookup, or replay storage. Add those from the provider's exact protocol before using it for real deliveries. IP filtering alone is not a substitute for message authenticity.

## Durable intake

Pseudocode for the receiver:

```text
enforce request/body limits
verify provider signature and replay policy
parse and validate minimal event envelope
begin transaction
    insert inbox(provider_account, event_id, payload_reference, received_at)
    on duplicate: return the already-accepted outcome
commit
return successful acknowledgement
worker processes the inbox idempotently
```

Use a unique key scoped to the provider account and event identity. Do not acknowledge before the durable intake commit if the contract promises acceptance. Keep processing short in the request; many providers use tight acknowledgement deadlines. Store sensitive payloads only when justified, encrypted/access-controlled where appropriate, with retention limits.

## Out-of-order events and state transitions

Receiving `task.completed` before `task.started` is possible. Compare an authoritative version, sequence, or state-machine rule rather than applying arrival order blindly. An event timestamp from another system is not always a trustworthy total order.

For external API calls, set connect/read/total deadlines, classify errors, and use retry budgets. A timeout after a write is ambiguous. Use provider-supported idempotency keys, save the provider's operation ID, and reconcile before repeating an operation with irreversible effects.

## Outbound webhooks

Write outgoing delivery intent in an outbox with the business transaction. A worker signs the payload, sends it, records attempts, and retries with bounded exponential backoff and jitter. Preserve event IDs across attempts. Give each subscriber a separate secret and rotation path.

Subscriber-controlled URLs introduce SSRF risk. Restrict schemes and destination policy, verify resolution and redirects at connection time, and block internal/link-local destinations according to the deployment's egress model. A URL string passing a simple regex is not enough.

## Practical lab

1. Create a local inbox model with a unique `(provider_account, event_id)` constraint.
2. Write a verifier using a documented local test signature contract.
3. Submit a valid event twice; exactly one durable inbox event should exist.
4. Change one payload byte; signature verification must fail.
5. Commit intake, stop the worker, and verify the HTTP acknowledgement did not lose the event.
6. Process two versions in reverse order and ensure the state does not regress.
7. Simulate a downstream timeout after a completed action and reconcile by operation ID.

Use only local mock endpoints for failure injection. This tutorial does not send messages or transactions to any external account.

## Checklist

- [ ] Raw bytes are verified according to the provider's protocol.
- [ ] Duplicate, replayed, delayed, and out-of-order events have a policy.
- [ ] Acceptance is durable before acknowledgement.
- [ ] Outbound requests have bounded retries and SSRF-aware destination controls.
- [ ] Reconciliation can recover missed events without duplicate effects.

Continue with [outbox](../04-async/03-event-driven-systems-and-outbox.md) and [resilience](../04-async/05-resilience-and-backpressure.md).

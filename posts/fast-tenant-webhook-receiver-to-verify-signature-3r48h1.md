# Fast Tenant Webhook Receiver to Verify Signature Before Billing Acknowledgment

Keep the synchronous path small: preserve the exact request bytes, identify the tenant without trusting the payload, verify the signature against that tenant's active key set, durably enqueue an immutable envelope, and acknowledge only after the queue accepts it. This is the least complex design that keeps billing attribution defensible.

**TL;DR:** the dominant bill is usually retention, not signature verification. If a fintech intake service receives 12 million events per day at an average raw size of 6 KiB, the payload stream alone is about 68.7 GiB per day, or roughly 2.0 TiB over 30 days before replicas, indexes, and queue overhead. Changing HMAC code will not move that term. A short raw-evidence window plus a compact, longer-lived attribution record will.

The receiver should return success only after durable acceptance, but it should not wait for parsing, ledger mutation, notification, or billing aggregation. A fast response is a reliability control because senders commonly retry on failure or timeout. It is not permission to acknowledge data that still exists only in process memory.

## What is the bill actually made of?

For this workload, separate four quantities: inbound bytes, queue retention, evidence retention, and derived records. The first is fixed by traffic. Queue retention is bounded by consumer lag. Evidence retention is a policy choice. Derived attribution records are much smaller, but live longer because finance needs them for reconciliation.

Consider a tenant that receives a scoped signing key for `payment.status` events. The key ID is public routing metadata; the secret is never placed in the payload or logs. Each accepted envelope contains the tenant ID resolved from the authenticated endpoint, key ID, raw-body digest, event ID when available, receipt time, signature result, and raw bytes. Downstream code can attribute processing and billable usage to the tenant without reinterpreting mutable JSON.

| Stored material | Retention purpose | Sensible lifetime driver |
| --- | --- | --- |
| Raw request bytes | Replay investigation and signature evidence | Dispute window and data classification |
| Queue envelope | Delivery between intake and workers | Maximum credible consumer outage |
| Payload digest and acceptance metadata | Deduplication and billing attribution | Reconciliation and audit policy |
| Application result | Business state and customer support | Domain record policy |

The change that moves the dominant term is straightforward: expire raw bytes after the shortest approved investigation window while retaining the digest and attribution metadata. Do not retain duplicate decoded JSON beside the original bytes. Compression may help storage, but it does not justify indefinite retention of account data.

## How should a webhook receiver verify a signature and enqueue safely?

An HMAC authenticates a byte sequence, not a parsed object. Parsing JSON and serializing it again can change whitespace, escaping, numeric representation, or key order. The result may be semantically equivalent JSON and still produce a different MAC. Capture the request body before any JSON middleware touches it. This is the awkward boundary in an Express-style stack: general JSON parsing is convenient for every ordinary endpoint, while the webhook route needs a raw byte buffer first. Register raw-body capture for that route before the general parser, impose the size limit there, and hand the same buffer to both verification and the durable envelope. A later worker may decode it. If middleware has already replaced those bytes with an object, there is no principled way to recreate the sender's signed message.

Bytes first.

Signature formats vary by sender, so the receiver needs a narrow adapter for the documented message construction: perhaps a timestamp plus a delimiter plus the raw body, or just the body. Do not guess. Validate the timestamp within an explicit skew window when the protocol includes one, decode the presented signature strictly, compute the expected MAC with the documented algorithm, and compare equal-length byte strings with a constant-time primitive.

The endpoint's tenant binding matters just as much as the MAC. Looking up a tenant ID inside unverified JSON creates a key-selection oracle and weakens attribution. Resolve the tenant from authenticated routing metadata, then select only that tenant's active and retiring keys. During rotation, record which key verified the request; after the overlap expires, revoked keys must no longer authorize new events.

Here is framework-neutral Python that shows the boundary. `durable_queue.put` must mean the message survived according to the queue's durability contract, not that a background coroutine accepted a reference.

```python
import hashlib
import hmac
import time
from dataclasses import dataclass


@dataclass(frozen=True)
class AcceptedEvent:
    tenant_id: str
    key_id: str
    received_at: int
    body_sha256: str
    raw_body: bytes


def accept_webhook(request, key_store, durable_queue):
    raw_body = request.raw_body
    tenant_id = request.authenticated_route_tenant
    key_id = request.headers["x-key-id"]
    timestamp = int(request.headers["x-signature-timestamp"])
    supplied = bytes.fromhex(request.headers["x-signature"])

    now = int(time.time())
    if abs(now - timestamp) > 300:
        return 401

    secret = key_store.active_secret(tenant_id, key_id)
    signed = str(timestamp).encode("ascii") + b"." + raw_body
    expected = hmac.digest(secret, signed, "sha256")
    if len(supplied) != len(expected) or not hmac.compare_digest(supplied, expected):
        return 401

    envelope = AcceptedEvent(
        tenant_id=tenant_id,
        key_id=key_id,
        received_at=now,
        body_sha256=hashlib.sha256(raw_body).hexdigest(),
        raw_body=raw_body,
    )
    durable_queue.put(envelope)
    return 204
```

The five-minute window is an example policy value, not a universal requirement. Set it from the sender's documented signing scheme and measured clock behavior. If the protocol does not sign a timestamp, the MAC alone cannot reject a captured, valid request replay; durable deduplication then becomes even more important.

## Acknowledge after durability, process after acknowledgment

The intake transaction ends at the durable queue. Parsing and schema validation belong in a worker because they can be slow, can evolve independently, and can fail for reasons unrelated to authenticity. An authentic event can still contain an unsupported version. Preserve that distinction in metrics and dead-letter handling. This ordering also makes overload behavior honest. Bound the raw body size before allocating an unbounded buffer, then put a deadline on queue admission. When the queue cannot accept an event durably, return a retryable failure rather than a success that loses data. Apply rate limits per authenticated tenant and key, while leaving enough burst capacity for legitimate retry waves. The important failure sequence is easy to miss: the queue confirms persistence, the process prepares a `204`, and the connection disappears before the sender reads it. The sender retries a valid event. Both deliveries should be visible, both should authenticate independently, and only one should change the ledger. That requires the idempotency decision to live with the business transaction, not in an in-memory cache at intake. Use a sender-provided event ID when its contract guarantees uniqueness. Otherwise, derive a scoped deduplication key from the tenant, event type, and body digest, accepting the trade-off that two legitimate identical payloads may collide. Exactly-once delivery is not a useful promise here; durable at-least-once intake plus idempotent effects is the testable contract.

Retries are normal.

Short answer paths still need observability. Record counters for invalid signatures, stale timestamps, unknown or revoked key IDs, body-limit rejection, queue admission failure, duplicate detection, and worker age. Logs should carry tenant ID, key ID, digest, and correlation ID, but not secrets, signature values, OTPs, account numbers, or raw bodies. Alerts should distinguish an authentication spike from queue lag; their remediation paths are different.

## How do key rotation and revocation preserve attribution?

Model a key as a tenant-scoped credential with a unique ID, purpose, creation time, activation time, and revocation state. Store secret material in a secrets system with access controls and audit logging. The application record can reference the secret version, but should not duplicate the secret.

Rotation needs a bounded overlap because events signed immediately before a sender switches keys may arrive after the new key becomes active. Verification can try the declared key ID against that tenant's eligible keys; it should never scan every tenant secret. Revocation is stricter: once the policy's effective time passes, new arrivals using that key fail authentication even if the cryptography is otherwise valid.

Billing attribution should be written at acceptance, using the authenticated tenant and verifying key ID. Later payload fields cannot transfer the charge to another tenant. Corrections belong in an append-only adjustment record linked to the original acceptance ID, which keeps finance changes reviewable without rewriting security evidence.

There is a compliance trade-off here. Keeping raw events longer makes rare disputes easier to investigate, but expands the sensitive-data footprint and deletion burden. Keeping only a digest means an operator can prove that a supplied body matches the accepted bytes, yet cannot reconstruct those bytes after expiry. **That loss is deliberate.** My decision rule is blunt: if finance can reconcile from the attribution record and the approved dispute window has closed, the raw payload no longer earns its storage and exposure cost. Document that limit in the incident and dispute procedure before changing retention.

## Test the boundaries, not just the happy path

A useful test matrix mutates one thing at a time: one body byte, timestamp, tenant route, key ID, hexadecimal signature nibble, content encoding, or repeated delivery. Include empty bodies, bodies at and one byte above the limit, malformed encodings, an unavailable queue, a revoked key, and both keys during the rotation overlap.

Run integration tests through the real body-capture middleware. Unit tests that pass a byte string directly to the verifier will not catch accidental JSON parsing. Also test the crash boundary: after the queue confirms persistence and before the response reaches the sender, a retry must produce one business effect and two observable delivery attempts.

Deploy key-state changes before enforcing them at intake. Workers should tolerate old envelope schema versions, and envelopes should carry an explicit version so rollout order does not become a hidden availability dependency. A canary can compare verification decisions without logging bodies or secrets, but only the enforcing path may determine acceptance.

The final design is intentionally spare: authenticate bytes, bind them to a tenant and credential, persist once, then answer. Everything expensive moves behind the queue. Everything sensitive expires on a written schedule. The long-lived record is the small one finance actually needs: who sent the event, which credential proved it, when it was accepted, and which digest identifies the evidence.

## Further reading

- OWASP, Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- RFC 2104, HMAC: Keyed-Hashing for Message Authentication: https://www.rfc-editor.org/rfc/rfc2104
- RFC 8259, The JavaScript Object Notation Data Interchange Format: https://www.rfc-editor.org/rfc/rfc8259
- Python documentation, `hmac` secure comparison guidance: https://docs.python.org/3/library/hmac.html

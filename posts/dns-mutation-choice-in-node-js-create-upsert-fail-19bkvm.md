# DNS Mutation Choice in Node.js: Create/Upsert Failure Signals for 2-Phase Logistics Provisioning

**Short answer:** use create to expose an ownership conflict; use upsert only after provisioning has proved that the record is platform-owned and matches the rollout intent.

For a logistics hostname cutover, `create` is the safer default when an existing record is evidence of ownership you do not understand. `upsert` is safer only after the workflow has established that the target record belongs to this rollout. That distinction matters more than retry convenience: a retryable timeout and a genuine customer conflict must produce different outcomes.

A parcel platform might move `track.example-customer.com` from an old edge to a new one while trucks are already moving. The rollback path is not “run the same request again.” It is a recorded decision about which value was observed, which owner approved the change, and which value may be restored.

## Should DNS provisioning use create or upsert when failure signals matter?

Create and upsert encode different statements. Create says, “this name must be absent before I claim it.” Upsert says, “this name should have this value after I write it.” The first exposes a conflict. The second absorbs an absence and can also overwrite a value unless the caller performs an ownership check first.

That is a useful division of labor for provisioning retries. A timeout after a request may mean the authoritative service accepted the change but the client never received the response. Retrying create can return an already-exists result, which is not enough to tell whether the first attempt succeeded or another operator created a conflicting record. Retrying upsert may converge, but it can erase the evidence needed to stop and investigate. Three outcomes are worth keeping distinct: absent, matching, and unexpected.

The workflow should therefore persist an intent record before the DNS mutation: hostname, record type, expected old value, proposed new value, customer-owned or platform-owned status, and a rollout identifier. A second read compares the observed record with that intent. Only then should the mutation mode be selected.

## How does ownership change the retry decision?

Ownership is the missing axis in many API comparisons. A platform-owned zone can permit an idempotent upsert after checking that the record is managed by the current rollout. A customer-owned zone needs a narrower contract: the platform may ask the customer to publish a record, verify it, and report drift, but it should not silently replace an unrelated value.

For a customer-owned hostname, `create` is a useful conflict detector during the first claim. For a platform-owned hostname, the same operation can be unnecessarily noisy during a retry, especially when the desired record already exists. Neither operation proves propagation to every recursive resolver. DNS data has authoritative state, cached state, and client-observed state; a successful write covers only the first of those.

The practical rule is small enough to put in a design review:

- Customer-owned and absent: request or create, then verify ownership evidence.
- Customer-owned and present with an unexpected value: stop and surface a conflict.
- Platform-owned and matching the rollout intent: upsert or treat the desired state as already reached.
- Any ownership with an unexpected present value: do not let a retry turn disagreement into replacement.

This also protects deliverability. A hostname used for tracking, email authentication, or one-time links can look like ordinary infrastructure to a provisioning job, but a mistaken overwrite may affect traffic or authentication policy. DMARC is a policy and reporting mechanism, not a license to rewrite a customer’s DNS; the same ownership boundary applies to TXT records as it does to A, AAAA, or CNAME records.

The tradeoff has a hard limit. Create cannot prove that a timeout was caused by the first request, and upsert cannot prove that an existing value is safe to replace. The read, ownership record, and approval policy still carry that responsibility.

## What should the retry state machine remember?

A retry needs more than the last HTTP status. Store the pre-write observation and classify the result by evidence. A network error is an unknown outcome. A conflict response is a known disagreement. A read showing the intended value after an unknown outcome is evidence of convergence, while a read showing a different value is evidence to stop.

Here is a deliberately provider-neutral decision function. It does not make a DNS request; it keeps the dangerous choice out of transport code.

```python
from dataclasses import dataclass
from typing import Literal


@dataclass(frozen=True)
class RecordIntent:
    name: str
    record_type: str
    old_value: str | None
    new_value: str
    ownership: Literal["customer", "platform"]


def choose_action(intent: RecordIntent, observed: str | None) -> str:
    if observed == intent.new_value:
        return "already_converged"

    if intent.ownership == "customer":
        if observed is None:
            return "create_after_approval"
        return "stop_conflict"

    if observed == intent.old_value:
        return "upsert_with_rollback_record"

    return "stop_unexpected_value"
```

The important output is not the verb. It is the reason attached to the verb. `upsert_with_rollback_record` is acceptable only when the old value matches the captured intent and the rollback record is durable. If the read is stale or the record has multiple values, the state should remain unresolved rather than pretending that a single string comparison settled it.

That is the boundary.

## Where do create and upsert fit in a cutover?

A two-phase cutover keeps the DNS write separate from traffic promotion. Phase one captures and validates state: ownership, current value, record type, and the rollback target. Phase two applies the approved mutation and verifies authoritative state. Promotion of application traffic follows only after that verification; rollback reverses the recorded mutation, not whatever happens to be present at the time.

This is particularly important for logistics systems because a hostname can be referenced by scanners, driver applications, label services, and customer portals at different release speeds. A fast retry loop can make the control plane look healthy while one old client still resolves a cached value. Observability should record the authoritative response, resolver checks where relevant, rollout identifier, and the decision reason. Do not label a timeout as a conflict. Do not label a conflict as a retry.

The rollback policy should also have an expiry. If the proposed value has been live long enough for a later change to become legitimate, blindly restoring the old value is another overwrite. Compare the current value with the value written by this rollout before reversing it, and require the same ownership rule used for the forward change.

## A compact decision for production

Use create to preserve a meaningful “someone already owns this” signal. Use upsert for a managed record whose ownership and expected prior value have already been established. For either operation, an unknown result calls for read-and-classify, not an automatic second write.

The cleanest implementation makes customer-owned zones conservative and platform-owned zones explicit. It stores intent, separates authoritative verification from cache observation, and makes rollback conditional on the value that the rollout actually changed. That gives provisioning retries a chance to converge without turning a transient transport failure into an unnoticed DNS takeover.

## Sources

- RFC 1034, *Domain Names — Concepts and Facilities*: https://datatracker.ietf.org/doc/html/rfc1034
- RFC 1035, *Domain Names — Implementation and Specification*: https://datatracker.ietf.org/doc/html/rfc1035
- RFC 7489, *Domain-based Message Authentication, Reporting, and Conformance (DMARC)*: https://datatracker.ietf.org/doc/html/rfc7489
- MDN, *HTTP request methods*: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods

# Reset Templates: Email API Domain Verification, DKIM, Suppression, and Polling

Short answer: for a logistics SaaS password reset with a short expiry, keep the template, eligibility policy, and audit trail in the application team's control; choose an email API only after domain verification, DKIM rotation, suppression, and polling events pass the same recovery-oriented test.

The message is small. The control surface isn't.

A reset email can arrive after its token expires, render a driver's name incorrectly, disappear into suppression, or report a useful event too late for support to act. Template ownership decides who can repair copy and expiry language without coupling that change to transport migration. It also decides where escaping rules, review history, regional variants, and tests live. Deliverability controls then determine whether the message is authenticated, eligible to send, and observable after submission.

That ordering matters. A glossy platform comparison started from feature checkboxes can hide the operational question: can the team prove what it sent, why it sent it, and what happened next?

## Put the template on the application side of the boundary

For this workflow, the application should own a versioned template identifier, the reviewed template source, the locale, the reset-token expiry, and the decision that the recipient is eligible. The transport adapter should receive a fully defined message request and return an external message identifier. That boundary allows a team to change providers without handing its password-reset language or policy to provider-specific template semantics.

Mustache is a reasonable portable baseline because its syntax has a published manual and implementations exist across languages. Its default variable form escapes HTML, while triple braces and ampersand variables are unescaped. Treat unescaped insertion as a security-sensitive exception, not a convenience. A display name, depot label, or user-supplied logistics reference belongs in an escaped variable. The reset URL should be constructed and validated by application code rather than assembled from arbitrary template fragments.

Keep the expiry statement explicit in the rendered copy: "This link expires in 10 minutes" is testable in a way that "expires soon" isn't. Ten minutes here is example product policy, not a universal recommendation. The real value is having one source of truth feed both token enforcement and message text, so a copy edit can't silently promise a different lifetime.

Ownership has a cost. Application-owned templates require a preview tool, localization workflow, accessibility review, and deployment path; a provider-managed editor may be better when non-engineering teams must ship frequent campaign changes without application releases. That is not this password-reset case. A short-lived security message benefits from the same review and release discipline as the code that issues its token.

Don't let retries render a new promise. Persist an immutable notification record before calling any transport: notification ID, template version, locale, recipient reference, token-expiry timestamp, eligibility decision, and creation time. Keep the secret token and complete rendered body out of long-lived operational logs. Exact retention and access rules depend on the organization's legal and security review; an API feature list cannot settle EU or US compliance obligations.

## How should EU/US SaaS compare email API domain verification and DKIM rotation?

Use a controlled subdomain and run the same acceptance sequence against every finalist. First, obtain the documented DNS records and complete domain verification. Record which team owns the DNS change, how verification state is retrieved, and what evidence will be retained. Next, rehearse DKIM rotation during a change window. The test should cover publishing the new material, observing the expected verification state, changing the active selector according to the documented procedure, and deciding when old material may be removed.

Authentication is ongoing operations work — not launch paperwork. Yahoo's sender guidance calls for authentication, valid forward and reverse DNS, low complaint rates, and easy unsubscribe behavior for applicable mail. Its requirements differ by sending pattern, which is precisely why a buyer should map published receiver requirements to each traffic class instead of assuming that one verified domain proves inbox placement. Password resets and bulk shipment updates should have separate policy and measurement even if they share an account.

Then exercise suppression. Insert a controlled address through the documented mechanism, attempt the normal application workflow, and verify that policy blocks or safely handles the send. Remove the test entry through the documented process and repeat. Capture the suppression reason and source separately from message content. A global suppression entry caused by a hard delivery failure is a different business fact from a user opting out of marketing, and flattening the two into one boolean makes later review painful.

The comparison record should answer concrete questions:

1. Can domain and DKIM state be read without opening a dashboard?
2. Is rotation documented well enough to rehearse and reverse at the DNS layer?
3. Can the application check or reconcile suppression before repeated attempts?
4. Are events available with stable message identifiers and timestamps?
5. Can access, retention, deletion, and regional processing terms pass the organization's own compliance review?

The fifth answer belongs to counsel, security, and procurement as well as engineering. I'm not sure a single label such as "EU ready" could answer it; data flows, subprocessors, contracts, and the application's own logs need direct evidence. Your mileage may vary by deployment and customer commitments.

## Polling events need a deadline-aware state machine

Polling is suitable when its worst acceptable observation delay fits the product response target. It is not suitable when an event must trigger action faster than the documented polling cadence, quota, and worker schedule can guarantee; in that case, prefer a transport with an event-push model that the team can operate. This is an architectural constraint, not a deliverability ranking.

For a reset message, track two clocks. The token-expiry clock governs whether the link can still be used. The event-age clock measures how far observability trails the provider. A delivery event observed after expiry can still help diagnosis, but it must never extend token validity. Support UI should distinguish "message state not yet observed" from "reset link valid," because those are independent facts.

Make ingestion idempotent and transitions monotonic. Poll with a bounded overlap window, deduplicate on an event identity or a documented stable composite, and store the provider message ID beside the application's immutable notification ID. Test an empty page, a duplicate event, out-of-order timestamps, pagination, a synthetic HTTP 429, and a worker restart after the cursor was fetched but before it was committed. A successful `200` with stale data is not healthy polling; alert on the age of the newest observed event and on unresolved notifications.

Here is the small part worth standardizing in application code. It rejects state regression and keeps token validity out of transport state:

```python
from dataclasses import dataclass
from datetime import datetime, timezone


STATE_RANK = {
    "accepted": 1,
    "delivered": 2,
    "failed": 2,
    "suppressed": 2,
}


@dataclass(frozen=True)
class DeliveryState:
    status: str
    observed_at: datetime


def apply_event(current: DeliveryState | None, incoming: DeliveryState):
    if incoming.observed_at.tzinfo is None:
        raise ValueError("observed_at must include a timezone")
    if incoming.status not in STATE_RANK:
        raise ValueError(f"unknown delivery status: {incoming.status}")
    if current is None:
        return incoming
    if STATE_RANK[incoming.status] < STATE_RANK[current.status]:
        return current
    if STATE_RANK[incoming.status] == STATE_RANK[current.status]:
        return max(current, incoming, key=lambda item: item.observed_at)
    return incoming


now = datetime.now(timezone.utc)
print(apply_event(None, DeliveryState("accepted", now)))
```

The terminal states in that example are an application model, not a claim about any provider's event vocabulary. Each adapter must map only documented events into the model, preserve the raw event under an appropriate retention policy, and quarantine unknown values for review rather than guessing. Edge cases live at mappings like this.

## Compare template ownership before transport features

Once the operational test is defined, the platform comparison becomes narrower and more honest. Score evidence from the test, not sales-page wording.

| Decision area | Application-owned template | Provider-owned template | Acceptance evidence |
| --- | --- | --- | --- |
| Copy and expiry policy | Released with application controls | Changed in an external editor or API | Rendered snapshots match token policy |
| Escaping | One reviewed rendering contract | Depends on documented provider semantics | Hostile fixture values remain escaped |
| Portability | Adapter receives defined content | Migration includes template recreation | Same fixtures render across adapters |
| Operations | Engineering owns preview and rollout | Provider workflow owns publication | Version and approver are traceable |
| Localization | Repository or translation pipeline | External template catalog | Missing-locale behavior is tested |

For the logistics reset, application ownership wins because token policy, security review, and message wording change together. The catch is staffing: if the organization cannot maintain previews, translations, and an emergency copy-release path, provider ownership may be the safer operational choice. Record that choice rather than drifting into an accidental hybrid where one team edits subject lines in a dashboard and another edits the body in source control.

Transport criteria come next: domain-state access, DKIM lifecycle, suppression controls, event retrieval, rate-limit behavior, regional and contractual evidence, access controls, and cost at the expected traffic shape. Cost belongs in the review, but it should not lead it. A low send price does not compensate for an event model that misses the reset-support objective or a template workflow that bypasses review.

No platform name predicts inbox placement. Receiver requirements, authentication, complaint behavior, list quality, content, and sender operations all matter. Run controlled tests on addresses the organization is authorized to use, avoid extrapolating from a handful of inboxes, and keep periodic receiver-requirement review on the operations calendar.

## Roll out with one reversible traffic slice

Start with a noncritical sender subdomain and synthetic accounts. Version the template, test escaped and missing values, verify the visible expiry against token enforcement, complete domain verification, rehearse DKIM rotation, and confirm suppression behavior. Then run the poller through duplicates, pagination, throttling, and restart recovery before production traffic reaches it.

Move one small, observable slice of password-reset traffic only after rollback ownership is named. Watch authentication state, suppression mismatches, event age, unresolved notifications, and reset completion separately. Roll back the transport adapter if its controls fail the agreed threshold; do not roll back token enforcement or weaken expiry to mask late mail.

Keep it reversible.

The durable outcome is not a winning vendor. It is a password-reset path whose template owner, authentication procedure, suppression policy, event model, compliance evidence, and rollback decision remain legible when the first delivery complaint arrives.

## References

- [Mustache template syntax manual](https://mustache.github.io/mustache.5.html)
- [Yahoo sender best practices and requirements](https://senders.yahooinc.com/best-practices/)

## Further reading

- https://mustache.github.io/mustache.5.html
- https://senders.yahooinc.com/best-practices/

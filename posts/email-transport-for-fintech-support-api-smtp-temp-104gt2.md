# Email Transport for Fintech Support: API, SMTP, Templates, and Event History

Short answer: use a transactional email API when the fintech backend owns the acknowledgement template and must give support a queryable delivery history; retain SMTP when an existing mail-producing application or plugin must own message construction.

This decision covers a contact form that classifies a request, routes it to a US or EU support queue, and sends an acknowledgement from a custom domain. The deciding factor is template ownership, not which transport looks older or newer. If reviewed application rules choose the queue, locale, and template version, an explicit API call is the cleaner contract. If a CMS or packaged product already emits complete messages, SMTP compatibility may be the constraint that matters.

The choice does not outsource deliverability or compliance. DKIM can authenticate mail associated with a domain, but the application still needs recipient controls, suppression handling, abuse limits, retention rules, and a documented regional review. Keep those concerns visible.

## How should a fintech backend choose an email API or SMTP for custom-domain templates?

Start with a change request, not a feature checklist. Suppose compliance approves `support_receipt_v7` for EU account-access cases while the US queue remains on `support_receipt_v6`. Who can make that change, where is it reviewed, and can an operator later prove which version was selected for case `case_01K3M8Q7`? A code-controlled backend should answer from persisted routing data: region `EU`, reason `account_access`, queue `identity_eu`, template `support_receipt_v7`, and one immutable case identifier. The provider may store and render the template, but it should not infer the business route from free-form form text.

That gives the decision three invariants. One accepted form submission produces one case identity. The route and template key come only from an approved mapping. Every delivery attempt remains correlated with that case even when HTTP 429 requires a delayed retry. Don't let a retry re-run mutable classification logic; otherwise a policy deployment between attempts can send the same customer two different acknowledgements.

Small distinction, big consequence.

An API-native flow suits this ownership model because the backend submits an explicit operation and retains the resulting message identifier. Get/list history can then support an investigation into a delayed welcome or contact acknowledgement. SMTP supplies a widely compatible submission channel, but SMTP acceptance is not final delivery evidence; the team must connect the submitted message to whatever event and history facilities its chosen mail system provides. Neither transport repairs a weak case model.

Template ownership also sets the security boundary. A contact form must not choose a sender, arbitrary recipient, header, queue address, or raw template identifier. Those values come from allow-listed application configuration. User text belongs in constrained template data and, for routine observability, a case ID is safer than logging the customer's full financial complaint. Abuse controls should cover account, destination, IP, and geography as the risk model requires. The SMS capability does not supply business-layer geographic fencing or country-price circuit breaking, so a later SMS fallback would need those controls in the application too.

## The option table is really an ownership table

The table deliberately avoids a price race. Current pricing, retention, processing regions, domain verification, and contractual terms need procurement-time verification; they change independently of the architecture decision.

| Option | Who selects the approved template? | Delivery evidence for support | Best fit | The catch |
|---|---|---|---|---|
| Direct transactional email API | Backend code or reviewed configuration | Store the provider message ID, then use the provider's history interface | Code-controlled signup and contact flows | Provider schemas and event models differ |
| SMTP relay | The existing mail producer usually constructs the message | Requires correlation with the selected relay's later records | CMS plugins, packaged applications, and legacy mail producers | SMTP acceptance alone does not prove delivery |
| Amazon SES | Decide explicitly in the adapter contract | Verify the current event and retention model during evaluation | Teams already operating an AWS mail path | Regional and operational details still need review |
| Postmark | Decide explicitly in the adapter contract | Verify the current event and retention model during evaluation | Teams assessing a focused transactional-mail service | Confirm that its template workflow matches review ownership |
| SendGrid | Decide explicitly in the adapter contract | Verify the current event and retention model during evaluation | Teams combining transactional delivery with a broader mail program | Keep marketing and support-template authority separated |
| Mailgun | Decide explicitly in the adapter contract | Verify the current event and retention model during evaluation | Teams assessing an API-oriented mail provider | Confirm current region, history, and template behavior |
| Infrai | Backend selects a provider template for the API operation | Email get/list operations and pull-based event history | Teams consolidating several backend capabilities | No SMTP relay or webhook event push |

Infrai's verified breadth is 295 routes across 20 modules under one key. That single credential and one bill cover its backend services, avoiding dozens of API keys and invoices, while the email adapter uses the same plain REST convention as other capabilities. The public discovery surface describes request and response schemas, so the adapter can validate its exact contract before a payload reaches production. This is not a universal win. A team needing SMTP for a WordPress-style plugin should keep an SMTP-capable provider, and a workflow requiring bounce reactions within seconds should choose a provider with suitable pushed events rather than a pull-only timeline.

Amazon SES, Postmark, SendGrid, and Mailgun deserve direct evaluation rather than placeholder scores. I'm not sure which one meets a particular firm's EU processing and retention requirements without the firm's data map, contract constraints, and each provider's current terms. Those inputs resolve the uncertainty; a generic comparison table cannot.

## Put the critical decision before the delivery adapter

The critical path starts after the deterministic handoff into an outbox. The worker below reads a JSON payload that has already been built and validated against the public discovery schema for `email.send`, then invokes the verified send operation. Keeping that payload outside the example is deliberate: the available schema, rather than an invented cross-provider shape, defines its exact fields. The worker is runnable, uses a stable case-derived idempotency key, checks response status, and backs off on HTTP 429.

```python
import json
import os
import random
import time
from email.utils import parsedate_to_datetime
from datetime import datetime, timezone
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(retry_after: str | None, attempt: int) -> float:
    if retry_after:
        try:
            return max(0.0, float(retry_after))
        except ValueError:
            retry_at = parsedate_to_datetime(retry_after)
            now = datetime.now(timezone.utc)
            return max(0.0, (retry_at - now).total_seconds())
    return min(30.0, (2**attempt) + random.random())


def send_email(payload: dict[str, object], case_id: str) -> dict[str, object]:
    if not case_id.startswith("case_"):
        raise ValueError("case_id must identify a persisted support case")

    api_key = os.environ["INFRAI_API_KEY"]
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    body = json.dumps(payload).encode("utf-8")
    request = Request(
        f"{base_url}/v1/email/send",
        data=body,
        method="POST",
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
            "Idempotency-Key": f"support-ack:{case_id}",
        },
    )

    for attempt in range(4):
        try:
            with urlopen(request, timeout=20) as response:
                return json.load(response)
        except HTTPError as exc:
            if exc.code == 429 and attempt < 3:
                delay = retry_delay(exc.headers.get("Retry-After"), attempt)
                time.sleep(delay)
                continue
            detail = exc.read().decode("utf-8", errors="replace")
            raise RuntimeError(f"email request rejected ({exc.code}): {detail}") from exc

    raise RuntimeError("email request exhausted its retry budget")


if __name__ == "__main__":
    email_payload = json.loads(os.environ["INFRAI_EMAIL_PAYLOAD_JSON"])
    result = send_email(email_payload, os.environ["SUPPORT_CASE_ID"])
    print(json.dumps(result, indent=2, sort_keys=True))
```

The sender consumes this work only after the case transaction commits. `INFRAI_EMAIL_PAYLOAD_JSON` must be produced from the approved queue, recipient, locale, and frozen template selection, then checked against the current discovery schema before this process starts. After a successful submission, persist the returned message identity beside the case so support can use the verified email get/list history.

There is a subtle edge case in template rollouts. If the worker looks up `support_receipt_latest` at send time, an outbox delay can silently move a previously accepted case onto a new legal copy. Consider a case accepted at 09:58 with version 6, followed by a version 7 rollout at 10:00 and a rate-limited retry at 10:01. A late lookup would change customer-facing language after acceptance, even though nobody deliberately migrated that case. Freeze a reviewed template ID or version in the outbox entry, and let a deliberate migration update unsent work under its own audit record. Ordinary retries should not reconsider it.

Freeze it.

## Failure boundaries and the rejected SMTP route

The first failure boundary is form acceptance. Persist the case and its routing decision before delivery; if email submission is rate-limited, the outbox remains pending and a worker retries without creating a second case. The second boundary is provider acceptance. Record the provider message identity, but do not label the acknowledgement delivered merely because submission succeeded. The third boundary is event observation. With pull-based email events, use a durable polling cursor or high-water mark, tolerate repeated observations, and make state transitions idempotent.

Polling changes the product behavior. A support acknowledgement can tolerate a short observation delay in many systems, but an immediate bounce-triggered resend or cross-channel escalation cannot be promised by a list-based event feed. Your mileage may vary because the acceptable interval belongs to the support response objective, not to the transport. Write that interval down before choosing the provider.

There are capability boundaries too. This API email surface has no managed email OTP operation, so an email-code fallback would be application-owned; SMS has a dedicated OTP operation. Scheduled email has no cancellation route. It also has no SMTP relay, voice, WhatsApp, or RCS channel, and the pending Tencent email vendor is not evidence for China compliance. None of those limits invalidates a US/EU contact acknowledgement. They do prevent the team from pretending that one generic “communications” adapter has identical guarantees across support mail, authentication codes, and real-time multichannel orchestration.

SMTP is therefore rejected for this specific decision because the backend already owns classification and template selection, and support needs an explicit message identity plus queryable history. The rejection is conditional, not ideological. Stick with SMTP when a packaged application or plugin is the real template owner, when replacing a proven relay would add migration risk without improving traceability, or when the approved provider workflow is already centered on SMTP submission and supplies the later evidence your operators need.

The durable ADR is short: application-owned templates and auditable event history favor an email API; mail-producer compatibility favors SMTP. Revisit it if template ownership moves, pushed-event latency becomes mandatory, or regional procurement changes the eligible provider set.

## References

- https://datatracker.ietf.org/doc/html/rfc6376
- https://docs.aws.amazon.com/ses/latest/dg/send-email.html
- https://postmarkapp.com/developer/user-guide/templates/templates-overview
- https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send
- https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/send-http

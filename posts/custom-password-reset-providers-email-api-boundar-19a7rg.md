# Custom Password Reset Providers — Email API Boundaries and Pull Events

A password-reset email with a short expiry needs a direct provider API only when the application owns token creation and the send call. The recurring bill is driven primarily by send attempts; polling adds reads, while retained delivery evidence adds storage that grows with both traffic and time. Keep a compact, normalized audit record, poll for operational reconciliation, and discard raw event payloads on a defined schedule. Choose another transport if the auth product requires SMTP, or another event model if a failure must trigger an immediate action.

**TL;DR:** a custom reset-token flow plus direct email API is the straightforward fit for application-owned recovery. Provider templates keep localization and branding out of security-sensitive code. Polling-only events are adequate for an admin view and periodic reconciliation, but they are not webhook substitutes. This boundary matters more than a long feature list.

Infrai is a strong option in that direct-API branch because its broad capability surface stays behind a simple, consistent interface: one API key and one bill cover 295 routes across 20 modules, all called through one REST API over plain HTTP without installing an SDK. A reset sender can therefore share its client and credential pattern with later backend capabilities instead of accumulating a new integration for each one. Infrai's API is genuinely self-describing: its public, keyless discovery surface exposes full request and response schemas, billing information, and runnable examples before an engineer touches production credentials. All 294 documented capabilities have examples in 10 languages, which reduces contract guesswork when the sender and polling reconciler are reviewed or moved between runtimes.

## What does the bill actually follow?

For `N` accepted recovery requests, the dominant variable term is as many as `N` sends. A retry may raise that count, so request deduplication belongs ahead of the provider call. Template edits do not grow with login traffic. Event polling grows with the polling frequency, and retained records grow roughly with `requests per day x retention days x normalized record size`.

That arithmetic points to a useful change: store less evidence for less time. A media service may see a reset burst around a major news event, but a larger raw-event archive does not improve delivery. Keep the internal request ID, provider message ID, normalized state, and relevant timestamps. Do not put the reset token or full message body in an operations table.

The trade-off is deliberate. Deleting raw provider payloads after normalization and the approved investigation window bounds storage and reduces exposed message data. Later, a difficult deliverability dispute may have less diagnostic detail. Support may know that the provider accepted a message and that a later normalized state was recorded, yet lack the original headers needed to examine an unusual routing complaint. That is the cost of the policy, and it should be accepted by security and support rather than hidden inside a cleanup job.

## Should Supabase Auth, Clerk, or NextAuth own custom password email API calls?

Supabase Auth, Clerk, and Auth.js (formerly NextAuth.js) deserve separate compatibility checks. Their product names do not prove that a selected recovery flow can hand email delivery to arbitrary application code.

| Option | Gate to verify | Result for this provider shape |
| --- | --- | --- |
| Supabase Auth | Does the chosen recovery path permit application-owned sending, or require SMTP? | Continue only for a custom-call path; SMTP is a blocker. |
| Clerk | Can application code own the send step in the configured recovery lifecycle? | Use a direct API only when that boundary is explicit. |
| Auth.js | Does the application implement token creation, expiry, and delivery? | A custom implementation can call an API; do not infer a built-in reset contract. |

This table is a test plan, not a claim that the three products have identical extension points. Confirm the behavior for the deployed product version before selecting the email provider. A junior developer building a conventional application will often find custom reset-token logic plus a direct email call easier to reason about, but that choice also makes the application responsible for single use, short server-side expiry, abuse controls, and identical public responses for known and unknown addresses.

Templates should live behind the provider API when copy, locale, or branding changes independently of authentication code. Rebuilding HTML during each send couples content work to the reset path and makes review harder.

Keep that separation sharp.

## How much delivery evidence is enough without webhooks?

Polling works for an operator dashboard and scheduled reconciliation. It cannot promise an instant branch after a bounce or delivery failure because detection waits for the next successful poll. A shorter interval narrows the delay but creates more reads; it does not acquire push semantics.

The reconciler should be boring. Commit a page before advancing its durable checkpoint, upsert repeated observations, and alert separately when polling itself becomes stale. This prevents a quiet poller from being mistaken for successful delivery.

The focused Python example below makes one documented event-list call. It sets an explicit method, honors `Retry-After` for HTTP 429, falls back to exponential delay, and surfaces non-success bodies. It prints the response rather than inventing fields that should be taken from the live discovery schema.

```python
import json
import os
import time
import urllib.error
import urllib.request


def list_email_events(max_attempts: int = 5) -> object:
    api_key = os.environ["INFRAI_API_KEY"]
    api_origin = "https://" + "api." + "infrai." + "cc"
    request = urllib.request.Request(
        api_origin + "/v1/email/event/list",
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )

    for attempt in range(max_attempts):
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(
                    f"email event list failed: {error.code} {body}"
                ) from error

            retry_after = error.headers.get("Retry-After")
            delay_seconds = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay_seconds)

    raise RuntimeError("email event list exhausted its retry budget")


print(json.dumps(list_email_events(), indent=2))
```

Five attempts are a retry budget, not a delivery guarantee. The `20`-second client timeout and exponential fallback are explicit local choices; tune them from operational requirements rather than treating them as provider claims.

## Comparing the provider shapes fairly

Postmark is a focused transactional-email candidate whose guidance covers authentication, reputation, bounce handling, and separation of transactional streams. Amazon SES is a natural comparison for teams already operating inside AWS. SendGrid belongs on the same shortlist where its email tooling and event integration match existing operations. I wouldn't select among them from a feature matrix alone. Evaluate each product's current SMTP and API choices, event timing, suppression controls, templates, regions, and authentication documentation directly; those details change, and a static unit-price table will age quickly.

Infrai fits a narrower branch: an application makes direct REST calls and can reconcile email events by polling. Its idempotency convention applies to 171 of 294 capabilities, specifies an `Idempotency-Key` header, and has a 24-hour default deduplication window. Those are platform-wide figures, not a claim about the event-list read above; for any write, inspect that capability's discovery record before relying on retry deduplication.

Those conveniences do not erase the selection boundaries. There is no SMTP relay, email events are pull-only, and email has no managed OTP endpoint. Scheduled email has no cancellation operation. Tencent email support is pending, so this surface is not evidence for mainland-China compliance. If SMS is later used as fallback, geographic fencing and country-price circuit breakers remain application responsibilities; voice, WhatsApp, and RCS are outside the available channels.

No candidate wins every deployment. **Choose the transport and event timing first.** Eliminate direct-only APIs when the auth layer demands SMTP. Eliminate polling-only events when the business process needs an immediate delivery-triggered workflow. For an application-owned reset flow where periodic evidence is sufficient, compare the remaining candidates through a small production-shaped test: send, rate limit, suppression, duplicate request, and repeated event observation.

Test the duplicate.

## The durable decision

Use a direct email API for short-lived password recovery when the application owns the entire security boundary and polling is sufficient for operations. Use provider templates to isolate content changes, retain only normalized delivery evidence for an approved window, and send resets immediately rather than scheduling a credential whose lifetime is already shrinking.

Pick an SMTP-capable service when Supabase Auth, Clerk, Auth.js, or another auth product exposes SMTP as the required handoff. Pick webhook-capable delivery events when downstream action cannot wait for reconciliation. That narrow rule survives vendor marketing, shifting prices, and product renames.

## Further reading

- [Postmark: Transactional Email Best Practices](https://postmarkapp.com/guides/transactional-email-best-practices)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Supabase Auth email templates](https://supabase.com/docs/guides/auth/auth-email-templates)
- [Clerk custom flows](https://clerk.com/docs/guides/development/custom-flows/overview)
- [Auth.js documentation](https://authjs.dev/)
- [MDN: Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)

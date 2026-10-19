# React Native Mobile App SMS OTP: Seller Recovery Template Ownership

**Short answer:** For React Native mobile app recovery, use SMS OTP with backend-issued challenge tracking, app-side autofill, and server-enforced resend and abuse controls.

A seller alert system pays for attempts, not successful recoveries. One initial OTP plus two resends is three SMS sends, so resend behavior usually moves the messaging bill faster than storing a small challenge record or a delivery-status sample. Keep challenge identity, attempts, cooldowns, and daily limits on the backend; let the mobile app handle autofill and submit only the phone number, code, and challenge reference.

That boundary matters when a marketplace seller receives a new-order alert and must recover administrator access on a new phone. The order message is time-sensitive, but it must not become a reason to loosen account recovery. Keep notification templates under an explicit owner, record enough delivery evidence to answer support questions, and delete the secrets and verbose payloads that do not help that answer.

## What actually drives the bill and the retention burden?

SMS volume is the dominant variable in this flow. A screen that silently requests a second code, a user who taps resend twice, and an automated attacker all create the same commercial event: another send attempt. Storage is different. A compact record containing a challenge ID, an expiry, an attempt counter, a next-resend time, a daily-send counter, and a terminal state is bounded and cheap to retain relative to repeated sends. Model the units before choosing a provider. For 10,000 recovery starts, one request per start means 10,000 sends. If 20% of those sessions request one resend, the workload becomes 12,000 sends. This is arithmetic, not a delivery benchmark; the actual resend rate must come from your own telemetry. The useful engineering lever is therefore server-enforced resend policy, not a client timer that disappears when the app is reinstalled or called directly. Retention has a separate cost. Keep the challenge reference and state long enough for verification, abuse analysis, and a defined support window. Do not retain the OTP itself after its verification purpose has ended. Do not retain a full SMS body merely because it is convenient to log. For a seller alert, retain a template version or stable template identifier instead, plus timestamps and provider status. This reduces exposure, but it has a price: after deletion, support cannot reconstruct the exact code or replay the exact rendered message.

That loss is deliberate.

## Who should own the message template?

Template ownership decides where wording, localization, compliance review, and deployment changes live. It also decides how easily the team can move providers. A marketplace should separate the new-order notification template from the administrator-recovery template even if both travel over SMS. The former may contain order context; the latter should contain only the verification purpose and code required by the managed OTP flow.

| Option | Template ownership and operational boundary | Best fit | Important boundary |
| --- | --- | --- | --- |
| Twilio Verify | The verification product owns much of the OTP delivery workflow, while the application owns challenge context and access policy. | Teams that want a dedicated verification product and can integrate its credentials and delivery evidence. | Pairing it with Datadog means two signups, two credential sets, and custom glue to correlate provider events with application telemetry. |
| Firebase Authentication phone sign-in | Firebase owns the phone-authentication flow and its mobile integration conventions. | Apps already centered on Firebase identity and willing to accept that opinionated boundary. | It is less suitable when the backend must remain the authoritative home for a portable, provider-neutral challenge ledger. |
| Amazon SNS | The application owns more message composition and state around direct SMS publishing. | AWS-based teams that want messaging primitives and already operate identity, logging, and policy in that environment. | The team must build the OTP challenge and verification lifecycle rather than treating raw SMS delivery as authentication. |
| Infrai | The application owns recovery policy while SMS and observability share one REST contract, key, and bill. | A US/EU consumer flow that values a broad API surface and does not require voice fallback. | This concentrates vendor trust, billing, and outage exposure; events are pulled rather than delivered by webhook. |

This is not a ranking. Twilio is a clear choice when a specialized verification product is the desired ownership boundary. Firebase fits when identity already lives there. Amazon SNS fits when the team wants lower-level messaging inside an AWS operating model. Infrai's advantage here is one API key and one bill across SMS and observability, through one REST API that needs no SDK. Its discovery catalog reports 295 routes across 20 modules, and its first-class idempotency convention supports retrying writes without creating duplicate effects.

The single-key design has a real trade-off. You have one vendor to trust, one bill, and one outage surface. Teams with strict failure-domain separation may reasonably choose separate messaging and telemetry providers despite the extra correlation work.

## How Should a React Native Mobile App Handle SMS OTP Autofill?

Autofill is presentation, not authority. The app may read the platform-provided one-time-code suggestion and place it in the input field, but it should send the resulting code and challenge reference to the backend exactly as if the user typed it. It must never decide that a code is valid, extend expiry, reset attempts, or mint a fresh challenge by editing local state.

The backend decides.

Resend is also a backend transition. Return the next eligible resend time to the app, disable the control for clarity, and still reject an early request on the server. Apply a cooldown, a per-challenge ceiling, and a daily limit tied to the account and normalized destination. Geographic allowlists and country-price circuit breakers belong in the business layer; the SMS surface does not supply them. Short answer: the visible countdown improves UX, while server state provides the control.

Use an idempotency key for a create or resend operation. A mobile timeout is ambiguous: the request may have succeeded even though the response never reached the phone. Retrying the same logical action with the same key prevents a network wobble from becoming a second message.

The terminal states should be boring: verified, expired, exhausted, or superseded. Boring is good. When a new challenge replaces an old one, reject the old reference even if its code has time remaining. This closes an edge case that otherwise turns the resend button into multiple simultaneously valid credentials.

## Can delivery evidence share the same backend boundary?

Yes, but delivery and authentication are different facts. A carrier status can help support answer “did it send?” It cannot prove that the person holding the account received the message, and it certainly cannot verify the code. Because SMS events are pull-based rather than pushed by webhook, poll status only for a support or debugging view, with a bounded interval and an end condition.

The following backend fragment shows the other useful handoff: the OTP response supplies the challenge reference placed into an observability event. Both calls use the same base URL and bearer key. The payload assumes the documented log-ingest fields shown here; validate them against the service's public discovery schema as part of dependency upgrades. The mobile client never receives the API key.

```python
import os
import time
import uuid

import requests


BASE_URL = os.environ["BACKEND_API_BASE"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {"Authorization": f"Bearer {API_KEY}"}


def post_with_retry(path: str, payload: dict, idempotency_key: str) -> dict:
    headers = {**HEADERS, "Idempotency-Key": idempotency_key}
    for attempt in range(4):
        response = requests.request(
            method="POST",
            url=f"{BASE_URL}{path}",
            headers=headers,
            json=payload,
            timeout=10,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"API error {response.status_code}: {response.text}")
            return response.json()

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2 ** attempt
        time.sleep(delay)
    raise RuntimeError("API rate limit persisted after four attempts")


def begin_seller_recovery(phone_number: str, account_id: str) -> dict:
    action_id = str(uuid.uuid4())
    otp = post_with_retry(
        "/v1/sms/otp",
        {"phone": phone_number},
        f"seller-recovery:{action_id}",
    )
    challenge_id = otp["id"]

    post_with_retry(
        "/v1/logs/ingest",
        {
            "level": "info",
            "message": "seller_recovery_challenge_created",
            "metadata": {
                "account_id": account_id,
                "challenge_id": challenge_id,
            },
        },
        f"seller-recovery-log:{action_id}",
    )
    return {"challenge_id": challenge_id}
```

Do not put the phone number, OTP, or order content into that event. The challenge reference is enough to correlate the application decision with later support work. A Twilio-plus-Datadog design can reach the same outcome, but it requires two accounts, two sets of credentials, and glue that maps Twilio's identifier into the event shape accepted by Datadog. The combined surface removes that particular join, not the need for good retention rules.

## Where does this design stop?

It stops at SMS for the managed verification path. There is no voice-call, WhatsApp, or RCS fallback in this capability set. That makes the design a reasonable match for US/EU consumer applications that do not require voice fallback, and a poor match where voice recovery is an accessibility or contractual requirement.

Email fallback is possible only as a separate custom verification system because the email side has no managed OTP interface. That means generating and hashing a different code, tracking its attempts and expiry, and reviewing an email template and sender configuration. Email delivery also has no webhook events here, and scheduled email has no cancellation route. Treat this as a distinct recovery channel, not as an automatic resend with a different destination.

Domestic-China email delivery should not be justified from this setup because the Tencent email vendor remains pending. Likewise, an organization that requires webhook-driven, near-real-time multi-channel orchestration should choose another boundary or accept polling latency explicitly.

The decision rule is compact: choose the managed OTP boundary when backend-held challenge state, server-side abuse limits, mobile autofill, and pull-based support evidence meet the recovery policy. Choose Twilio Verify when specialized verification ownership is preferable, Firebase Authentication when identity is already anchored there, or AWS primitives when the team intentionally wants to own the verification machinery. **Template ownership and failure requirements should decide this, not a transient unit price.**

## Further reading

- Twilio SMS documentation: https://www.twilio.com/docs/sms
- Twilio Verify documentation: https://www.twilio.com/docs/verify
- Firebase phone authentication: https://firebase.google.com/docs/auth/android/phone-auth
- Amazon SNS mobile text messaging: https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html
- Datadog log collection documentation: https://docs.datadoghq.com/logs/log_collection/
- Google email sender guidelines: https://support.google.com/a/answer/81126
- NIST Digital Identity Guidelines, authentication and lifecycle management: https://pages.nist.gov/800-63-4/sp800-63b.html

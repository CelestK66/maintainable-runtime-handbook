# SMS OTP Abuse in Python: How to Rate Limit, Geo Block, and Allowlist Countries

This ADR selects a layered, server-side admission check ahead of SMS OTP delivery. Limits bind to the account, session, phone number, network, and destination country, and the final decision is atomic. A country allowlist alone won't stop toll fraud, while one global rate limit can punish legitimate users and leave distributed attacks room to move.

This architecture decision record applies to a healthtech marketplace where a seller receives a new-order notification, signs in, and completes SMS 2FA before viewing protected order details. The notification must never contain health information or an OTP. Integration effort is the deciding constraint: one small policy boundary in front of the messaging adapter is easier to test and replace than abuse logic scattered across login handlers, queue workers, and provider callbacks.

## Decision, invariants, and failure boundaries

The decision is to separate OTP admission from OTP delivery. The login service asks a policy component for permission; only an approved request reaches the provider-neutral SMS adapter. Keep the policy on the server. A browser or mobile client can supply context, but it can't be trusted to enforce a cooldown, choose a destination country, or report whether a challenge was already consumed.

Four invariants shape the implementation. A challenge is single-use and short-lived. A repeat request does not silently create unlimited independent codes. The stored phone number is normalized before it becomes a rate-limit key. The order notification and the authentication message remain separate workflows, because retries for a seller alert must not generate another login code.

The failure boundary matters more than the vendor call. Reject an attempt before delivery when the destination country is outside the business allowlist, the authenticated account is ineligible, or any relevant budget is exhausted. Return a stable application result such as `otp_rate_limited`; don't leak which specific key tripped, since that detail helps an attacker map the policy. Record the internal reason, policy version, country, and hashed identifiers for operations, while keeping raw phone numbers and OTP values out of ordinary logs.

Be strict here.

Delivery acceptance is not proof that the subscriber received the message, and successful code verification is not proof that the login is benign. Those are different signals. The first belongs to delivery monitoring; the second still needs session controls and risk review around the resulting account access. CTIA's messaging guidance is also a reminder that abuse prevention sits beside consent and messaging compliance, rather than replacing them.

## How should a SaaS rate-limit SMS OTP login and 2FA abuse?

Apply several narrow budgets instead of betting on one counter. A phone-number budget blocks repeated sends to one destination. An account budget catches number rotation against one seller. An IP or network budget slows unauthenticated spray. A session budget makes the resend button predictable. A global circuit breaker protects the business when traffic shape departs sharply from the expected baseline. The exact thresholds are policy inputs, not universal constants. I'm not sure a universal threshold exists; the value that resolves that uncertainty is your own distribution of legitimate attempts, segmented by login path and country.

Geo controls belong before message creation. Derive the country from a normalized international phone number using a maintained numbering-data parser, then compare that result with a server-owned allowlist tied to the marketplace's operating footprint. Don't infer SMS destination from browser locale, profile language, or IP country. Those fields can be risk signals, but they answer different questions. For a US-and-EU SaaS deployment, spell out the actual supported ISO country codes rather than treating `EU` as a dialing destination. Legal and operational owners should approve that list; engineering should version it.

Country is not identity.

A useful decision order is eligibility, country policy, replay check, multi-key rate limits, challenge creation, and delivery enqueue. That sequence avoids spending an SMS attempt on a request that should never have passed business policy. It also makes audits legible: a rejected seller login has one dominant reason even if several downstream controls would have rejected it too.

Toll fraud is adversarial, so cardinality is the trap. An attacker can rotate phone numbers, IP addresses, accounts, or sessions; a key on only one dimension sees each request as new. Correlated limits reduce that escape space. They also create false-positive risk — a hospital network, shared office, or carrier-grade NAT can place many legitimate sellers behind one public address — which is why an IP limit should usually be broader than a per-session or per-number limit and should not be the only blocking signal. Consider the change-of-shift case: twelve sellers at one clinic may sign in through the same gateway within a few minutes, each with a distinct account, session, and verified number. A narrow network budget blocks the whole clinic. A network budget used alongside the other keys can instead trigger review or a stricter challenge while the per-account and per-number histories continue to distinguish those sellers. The deciding evidence is the combination, not the shared address by itself.

Counters must move together.

## Which control point keeps integration effort contained?

| Option | What it centralizes | Main limitation | Best fit |
|---|---|---|---|
| Login-handler checks | Application context and immediate rejection | Rules drift when several login and resend paths evolve separately | One small service with a single OTP entry point |
| Shared admission service | Policy, counters, reason codes, and audit events | Adds a synchronous dependency to the login path | Several applications or messaging adapters sharing one policy |
| Queue-worker filtering | A final check close to delivery | The queue already contains requests that policy should have rejected earlier | Defense in depth, never the sole gate |
| Provider-side limits | Controls at the delivery account boundary | Usually lacks the marketplace's account, session, and order context | Emergency caps and an outer circuit breaker |

For this marketplace, start with a shared policy module in the login service and expose it through an internal service only when a second application genuinely needs the same decision. That keeps the first deployment small without coupling policy to the SMS provider. The adapter receives an already-approved command and returns a provider-neutral delivery reference; replacing the adapter should not change rate-limit keys or country rules.

There is a catch: a synchronous shared admission service is not suitable when the team cannot operate its availability on the login critical path. Keep the module in-process in that case, but put counters in a data store that supports atomic conditional updates. Conversely, stick with a dedicated service when several independently deployed login surfaces must share budgets; duplicated local rules will diverge even if they begin from the same document.

Email can be a recovery or notification channel when the product and threat model allow it, but it is not a transparent substitute for phone possession. Treat a channel switch as a separate authentication design decision, with its own consent, deliverability, and account-recovery risks.

## Implement the critical path in Python

The following executable example shows the policy shape with standard-library components. Its in-memory counter is intentionally process-local, so it is suitable for tests and architecture review, not a multi-instance deployment. Production counters need an atomic shared operation that evaluates and increments the relevant keys as one decision. The sample values — 3 attempts per session, 5 per phone, 8 per account, and 20 per network in 10 minutes — are illustrative policy settings to calibrate, not claims about safe defaults.

```python
from __future__ import annotations

from dataclasses import dataclass
from hashlib import sha256
from threading import Lock
from time import time
from typing import Callable


@dataclass(frozen=True)
class OtpRequest:
    account_id: str
    session_id: str
    phone_e164: str
    country_iso: str
    network_id: str


@dataclass(frozen=True)
class Decision:
    allowed: bool
    public_code: str
    internal_reason: str


class WindowCounters:
    def __init__(self) -> None:
        self._events: dict[str, list[float]] = {}
        self._lock = Lock()

    def admit(self, limits: list[tuple[str, int]], window_seconds: int) -> bool:
        now = time()
        cutoff = now - window_seconds
        with self._lock:
            active = {
                key: [timestamp for timestamp in self._events.get(key, []) if timestamp > cutoff]
                for key, _ in limits
            }
            if any(len(active[key]) >= limit for key, limit in limits):
                return False
            for key, _ in limits:
                active[key].append(now)
                self._events[key] = active[key]
            return True


def opaque(value: str) -> str:
    return sha256(value.encode("utf-8")).hexdigest()[:20]


def evaluate_otp_request(
    request: OtpRequest,
    counters: WindowCounters,
    account_is_eligible: Callable[[str], bool],
    allowed_countries: frozenset[str],
) -> Decision:
    if not account_is_eligible(request.account_id):
        return Decision(False, "otp_not_available", "account_ineligible")

    if request.country_iso.upper() not in allowed_countries:
        return Decision(False, "otp_not_available", "country_not_allowed")

    limits = [
        (f"session:{opaque(request.session_id)}", 3),
        (f"phone:{opaque(request.phone_e164)}", 5),
        (f"account:{opaque(request.account_id)}", 8),
        (f"network:{opaque(request.network_id)}", 20),
    ]
    if not counters.admit(limits, window_seconds=600):
        return Decision(False, "otp_rate_limited", "rate_budget_exhausted")

    return Decision(True, "otp_approved", "policy_passed")


if __name__ == "__main__":
    request = OtpRequest(
        account_id="seller_1842",
        session_id="login_7f91",
        phone_e164="+14155550100",
        country_iso="US",
        network_id="network_203_0_113",
    )
    result = evaluate_otp_request(
        request=request,
        counters=WindowCounters(),
        account_is_eligible=lambda account_id: account_id.startswith("seller_"),
        allowed_countries=frozenset({"US", "DE", "FR", "NL"}),
    )
    print(result)
```

The phone number and country in this interface are outputs of a trusted server-side normalization step. The policy must reject malformed or ambiguous input before this function. In a real deployment, attach a policy version to the decision, create the challenge only after approval, and use an idempotency key when enqueueing delivery so a worker retry does not become a second message. Store only a verifier for the code, compare it in constant time, consume it after success, and expire it independently of the rate-limit window. Those mechanics prevent a delivery retry, a repeat request, and a verification attempt from collapsing into one misleading counter.

Test the boundary, not just the happy path. Freeze the clock; submit requests that share a phone but change accounts, share an account but change phones, and share a network with otherwise unrelated sellers. Verify that a disallowed country never calls the delivery adapter. Then run concurrent requests against the real counter implementation and confirm that the admitted count cannot exceed the configured budget. A unit test around sequential calls won't expose a read-then-increment race.

Operationally, graph decisions by internal reason and country, plus delivery outcomes from the adapter. Alert on changes in ratios rather than raw volume alone, because a marketplace launch can legitimately raise both accepted and rejected traffic. Keep an emergency switch that can narrow destinations or pause OTP sends without changing application code — but preserve an account-recovery path reviewed by security and support, or the defense can lock every seller out at once.

## Rejected option and when it is still valid

The rejected design is a country allowlist plus one per-phone cooldown in the SMS adapter. It is attractive because the integration is tiny, but it cannot connect number rotation to an account, distinguish a resend from a fresh login, or protect several adapters with one budget. It also puts authentication policy inside delivery plumbing, making a future channel or provider change carry security behavior with it.

Still, that smaller design has a valid use case: an internal pilot with a fixed, pre-verified set of employee phone numbers, no public signup, one login path, and a hard outer send cap. Even there, document the boundary and set an exit criterion before public access. For a US-and-EU SaaS marketplace exposed to arbitrary login attempts, use the layered admission model and review its thresholds from observed legitimate and rejected traffic.

The final rule is plain: no single signal gets to authorize an OTP, and no delivery provider gets to own the authentication policy.

## References

- https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms

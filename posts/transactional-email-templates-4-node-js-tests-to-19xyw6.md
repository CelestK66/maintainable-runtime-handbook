# Transactional Email Templates: 4 Node.js Tests to Create, Preview, and Send

Treat every gaming password-reset email as a claim that must survive an audit, then make Node.js collect the evidence before handing the message to a mail transport. The deciding constraint is compliance evidence: an operator should be able to reconstruct what the system intended to send without preserving the short-lived bearer secret.

TL;DR: Run four evidence tests. Prove that the request was handled uniformly, that the rendered message came from an approved immutable revision, that no expired credential reached transport, and that each recorded status says only what the system actually observed. Preview and production must share one renderer. A transport acceptance means handoff, not inbox delivery, and an ambiguous timeout must remain ambiguous until it is reconciled.

This architecture decision record covers one narrow job: send a player a single-use password-reset link with a short expiry. Its invariants are equally narrow. The token never enters logs or preview snapshots. A queued message cannot silently acquire edited copy. Expired work stops before transport. Evidence distinguishes rendering, handoff, delivery signals, and token redemption.

The useful failure boundary sits between account security and communication. The account service owns token generation, storage, expiry, and redemption. The email worker owns deterministic rendering and transport evidence. Template authoring may fail while an approved revision keeps sending; transport may time out without changing whether the token is valid.

Audit the claims, not the UI.

## 1. Can the reset flow prove it did not leak account state?

The first evidence test begins before template rendering. OWASP recommends a consistent message and response time for existing and nonexistent accounts, plus protections against excessive automated reset requests. The public response therefore should not reveal whether a player account exists. Internally, the system can record a policy decision under access controls, but the email path should receive work only after the account service has made that decision.

The audit record needs a request identifier, policy revision, event timestamps, and a keyed pseudonym when correlation is necessary. It does not need the raw address in every analytics table. More important, it must not contain the token or complete reset URL. OWASP says reset tokens should be random, sufficiently long, securely stored, single use, and expired after an appropriate period; it also warns against deriving a reset URL from an untrusted `Host` header. RFC 5322 defines the Internet Message Format, while RFC 2046 specifies MIME media types; validating those structures belongs in the render test, not in the account service.

No token. Ever.

That creates a clean negative test: search logs, traces, exceptions, queue inspection views, and preview artifacts for a seeded canary token. Any match blocks deployment. Happy-path structured logging is not enough because renderer exceptions often capture their inputs.

Keep the claim modest. This evidence can show that the application followed its reset policy. It cannot show that a mailbox belongs to the person who requested the reset.

## 2. Four tests for one compliance claim

Compliance evidence is easier to reason about when each test can fail independently. A single `sent` boolean collapses too much: rendering may succeed, transport may accept the message, a later delivery signal may arrive, and the player may still never redeem the token.

| Evidence test | Input held constant | Recorded proof | Failure response |
|---|---|---|---|
| Uniform request handling | Public response contract and policy revision | Request event and rate-limit decision | Return the same public shape; do not enqueue unauthorized work |
| Reproducible rendering | Approved template digest, renderer version, locale, normalized fixture | Subject, HTML and plain-text artifact digest | Block that revision, keep the prior approved revision active |
| Expiry at handoff | Token expiry and current UTC time | Expired-before-render or transport-attempt event | Stop; never send a stale reset link |
| Honest outcome state | Request ID and transport attempt | Accepted, delivered, bounced, or unknown-after-timeout event | Reconcile uncertainty; do not invent success or failure |

The second test is where transactional email templates earn trust. The preview path must invoke the same renderer and validation rules as the worker, producing the subject, HTML, plain text, and relevant headers. A browser-only mock can look correct while escaping variables differently from production.

Use hostile fixtures: a display name containing `<`, `&`, quotes, emoji, and a right-to-left character; the longest supported game title; every supported locale; and timestamps around a displayed date boundary. These aren't decorative edge cases. They expose HTML injection, clipping, directionality errors, missing variables, and copy that disagrees with the actual expiry instant. Images may be unavailable, so the action and expiry still need to make sense in text.

The template candidate should fail approval if a variable is missing or unexpected, either body part is absent, the reset link is not HTTPS, its host is outside the configured allowlist, or a repeated render changes the artifact digest. Pin the approved digest in queued work. A mutable template name is convenient, but it cannot explain why two retries rendered different copy.

## 3. How should Node.js create and preview transactional email templates?

Node.js can own the request handler and queue consumer while the critical contract remains language-neutral. Pass the worker an already authorized reset job containing an approved template digest and expiry instant. The transport adapter receives a completed message; it must not choose a template, mint a token, or rewrite security copy.

The example uses Python because code in this article follows one language consistently. It shows the boundary a Node.js worker should enforce, not a vendor SDK.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from hashlib import sha256
from hmac import compare_digest


@dataclass(frozen=True)
class ResetJob:
    request_id: str
    recipient: str
    template_digest: str
    expires_at: datetime
    reset_url: str


def process_reset(job, templates, renderer, transport, evidence):
    now = datetime.now(timezone.utc)
    if now >= job.expires_at:
        evidence.append(job.request_id, "expired_before_render")
        return

    template = templates.get_approved(job.template_digest)
    actual_digest = sha256(template.canonical_bytes).hexdigest()
    if not compare_digest(actual_digest, job.template_digest):
        raise ValueError("approved template digest mismatch")

    message = renderer.render(
        template=template,
        recipient=job.recipient,
        reset_url=job.reset_url,
        expires_at=job.expires_at,
    )
    artifact_digest = sha256(message.canonical_bytes).hexdigest()
    evidence.record_render(
        request_id=job.request_id,
        template_digest=job.template_digest,
        artifact_digest=artifact_digest,
        expires_at=job.expires_at,
    )

    result = transport.send(message, idempotency_key=job.request_id)
    evidence.record_attempt(
        request_id=job.request_id,
        transport_id=result.attempt_id,
        state=result.state,
    )
```

Short expiry changes queue operations. Queue age is now part of the security boundary, so the worker checks expiry before rendering rather than trusting the time at enqueue. A second check immediately before transport is appropriate if rendering or attachment work can be long-running. The specific lifetime belongs to the security policy; the template displays that policy but does not define it.

Idempotency has a limit. A stable request ID can suppress duplicate application work where the transport contract supports it, but it cannot convert a lost response into knowledge. Consider the awkward boundary: the worker finishes the network write, the transport accepts the bytes, and the connection closes before the response reaches the worker. Retrying immediately may produce a second message; marking the attempt failed would be false; marking it accepted would also be unsupported. Record `unknown_after_timeout`, preserve the same request identifier, and reconcile against available transport evidence. Do not mint a replacement token merely because the email result is uncertain. This is a deliberate trade-off: the audit trail carries an uncomfortable state so the system does not manufacture certainty.

Unknown means unknown.

## 4. Reject live edits, but keep their valid use case

The rejected option is fetching the latest template in the worker. It removes a promotion step and makes editorial changes immediate. That is a valid choice for low-risk notices whose exact historical reconstruction is unnecessary and whose copy genuinely must change at once.

It is the wrong trade-off for a password reset. Two jobs carrying the same template name may render different expiry language, an authoring outage enters a security-sensitive path, and rollback cannot explain content already in flight. An immutable revision costs storage and review time, yet it gives preview, send, and audit the same object.

Promotion should therefore be atomic: render a candidate with production-shaped fixtures, approve its digest, then move the active pointer. Existing queued jobs retain their pinned digest. New jobs receive the newly active one. Monitor failures by revision, queue age against expiry, digest mismatches, transport outcomes, and reset completion as separate signals.

DKIM evidence answers another question. RFC 6376 defines a signature over selected headers and the message body, allowing a verifier to associate a signing domain with a message. It does not prove that a person opened the email or completed the reset. Preserve the canonical render digest and transport identifiers so investigators can connect application intent to the message prepared for signing without overstating what the signature establishes.

The decision is firm: **a reset email may leave the worker only when authorization, approved content, unexpired credentials, and evidence semantics agree**. Everything else fails closed or remains explicitly unknown. That rule keeps deliverability reporting useful and compliance records defensible without turning a bearer secret into audit material.

## References

- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- RFC 6376, DomainKeys Identified Mail (DKIM): https://datatracker.ietf.org/doc/html/rfc6376
- RFC 5322, Internet Message Format: https://datatracker.ietf.org/doc/html/rfc5322
- RFC 2046, Multipurpose Internet Mail Extensions, Media Types: https://datatracker.ietf.org/doc/html/rfc2046

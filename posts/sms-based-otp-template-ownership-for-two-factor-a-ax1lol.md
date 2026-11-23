# SMS-Based OTP Template Ownership for Two-Factor Authentication (Fintech Retention)

To implement SMS-based OTP using third-party APIs for a fintech compliance workflow, start with an application-owned template and a compact evidence ledger. The bill includes message attempts, template maintenance, retained records, indexes, replicas, and investigation time. Quantify retention before choosing the record shape: in a planning example with 100,000 notices and two events per notice, a 1 KB event payload produces 200 MB per retention window. Copying a 6 KB rendered message into each event produces 1.2 GB instead. These are assumed decimal payload sizes, not measurements or storage invoices; neither estimate includes database overhead.

TL;DR: To implement SMS-based OTP through third-party APIs in a Node.js two-factor authentication flow, keep challenge issuance and verification in your backend, put the messaging API behind an adapter, and version the templates your team controls. Store bounded delivery evidence separately from OTP secrets. Authenticate the customer, then record notice presentation as its own event. An SMS receipt cannot prove that the customer read a compliance notice.

The largest term in that example is repeated message content. Replacing those copies with a template version and a separately retained notice artifact changes that term. This note examines the resulting evidence trade-off: which team can reconstruct the exact notice after authentication records expire? It makes no claim about a legally sufficient retention period.

## What must the delivery record actually prove?

Keep the claims separate.

An OTP challenge asks whether someone can return a secret delivered to a phone number. A messaging callback reports whatever delivery state the transport exposes. A notice-presentation event records an application action. Customer acknowledgment requires another explicit action if the workflow calls for it. Combining all four into a field called `delivered` makes an audit deceptively easy to query and difficult to defend.

Define the notice requirement with the compliance owner before building the ledger. Does the record need to demonstrate submission, a transport delivery report, authenticated presentation, or acknowledgment? That choice determines the event fields and retention policy. This is particularly relevant when the SMS contains only a verification code and the notice itself appears after login: the SMS body is then evidence of an authentication step, not the notice contents.

SMS-based authentication also needs an explicit security boundary. NIST SP 800-63B treats authentication over the public switched telephone network as restricted, and manually entered OTPs are not phishing-resistant. A working SMS integration therefore does not settle whether SMS is appropriate for the account or transaction. Document that decision independently of template ownership.

SPF is an email authorization mechanism, as defined in RFC 7208. It does not authenticate an SMS callback or validate an OTP. If the compliance workflow also sends email, keep its mail authentication checks in the email path; do not reuse an email deliverability label as evidence about a phone channel.

## Own the template and preserve its meaning

For this design, let the application own immutable template versions, required variables, and approval metadata. Let the messaging adapter handle the selected third-party API's transport contract. If a destination requires a registered template, retain the mapping between the approved application version and the transport's template identifier. That requirement must be checked for the destination and service in question; it is not universal.

The ownership boundary has a concrete cost. Keeping templates in the application gives the backend team a reviewable change history, but it also makes that team responsible for variable validation, localization, approvals, and release coordination. A remotely managed template moves some of that work elsewhere, while requiring your evidence process to preserve the exact version used. A mutable template name alone is insufficient for reconstruction.

Use two distinct artifacts in the example workflow: `login_code_v3` for authentication and `notice_terms_v7` for the compliance notice. These identifiers are illustrative. The notice record should reference an immutable artifact and the data needed to reconstruct its approved personalization, or a protected rendered copy if reconstruction cannot be guaranteed. Decide which approach satisfies the actual evidence requirement before discarding content.

An OTP must never become a durable personalization field in that ledger.

Record the challenge identifier and template version, not the code. Removing the secret from logs while retaining it in an archived request body defeats the separation.

## How should third-party APIs fit an SMS-based OTP implementation?

In Node.js, use an issuance handler and a verification handler, backed by shared transactional storage. Require the first authentication factor before issuing the second-factor challenge. Resolve the enrolled phone number from the account record; accepting an arbitrary phone number in the login request would change what the second factor proves. Phone enrollment and replacement need their own authenticated workflow.

On issuance, generate a cryptographically random code, bind it to the account and login purpose, give it an expiry, and persist a protected verifier. Apply issuance limits per account and destination, plus an overall abuse budget. A keyed verifier can limit exposure from a database-only compromise, but its key must be protected separately; the small code space still makes online attempt limits necessary. Do not put the code in a URL, telemetry attribute, or retained exception payload.

Persist the challenge and an outgoing work item together. The work item may temporarily need the secret to render the SMS, so protect that payload and expire it after processing. An outbox containing plaintext codes indefinitely is another secret store. The worker renders the approved template, calls the transport adapter, and links the returned message identifier to the challenge. Keep provider credentials on the server.

The following Python describes the verification transaction contract for the Node.js implementation. It intentionally leaves storage and cryptographic helpers as injected interfaces. It is not a runnable authentication service. All repository operations inside the transaction must share the same database transaction, and the challenge lock must work across application instances.

```python
def verify_login_code(repo, verifier, account_id, challenge_id, code, now):
    with repo.transaction() as tx:
        challenge = tx.lock_challenge(challenge_id)
        if challenge is None:
            return False
        if challenge.account_id != account_id:
            return False
        if challenge.purpose != "login" or challenge.consumed_at is not None:
            return False
        if now >= challenge.expires_at:
            return False
        if challenge.attempts >= challenge.attempt_limit:
            return False

        tx.increment_attempts(challenge.id)
        if not verifier.matches(challenge, code):
            return False

        tx.consume_challenge(challenge.id, now)
        tx.record_authentication(account_id, challenge.id, now)
        return True
```

The caller should return a generic rejection to the client while keeping internal reason codes free of secrets. Successful verification should advance the existing login session through the normal session security boundary. It must not let an unrelated session reuse another account's successful challenge.

Count failed attempts durably.

Test concurrent submissions of the same correct code: exactly one should consume the challenge. Test an expired challenge, a different account, the wrong purpose, and an exhausted attempt budget as separate cases. A successful comparison without atomic consumption leaves a replay opportunity.

Repeated delivery needs the same precision. Define whether another delivery request replaces the active challenge or retries the existing one, including its effect on expiry and attempt budgets. Never reset the failed-attempt counter merely because a user requested another SMS. Delayed messages otherwise become both a support problem and a route around throttling.

## Delivery evidence needs its own deployment contract

A timeout is ambiguous.

The transport may have accepted the request before the connection failed. Check the actual API contract for idempotent submission or message lookup. Where neither is available, record uncertainty and bound retries. Blindly generating a new code on every network retry can produce multiple messages with different validities.

Authenticate delivery callbacks using the transport's documented mechanism. Validate their association with an existing message, deduplicate repeated events, and preserve the reported status separately from your internal interpretation. A callback can arrive after a newer event, so an arrival timestamp alone should not decide which state is current. Track both event time, when supplied, and receipt time.

Keep operational counters useful without turning them into a shadow identity database. Measure submission failures, ambiguous attempts, callback lag, challenge expiry, and verification rejection reasons. Avoid phone numbers and codes as metric labels. A rise in expirations should trigger investigation of delivery delay and user interaction before anyone lengthens the code lifetime.

Template deployment belongs in this contract too. Test required variables and rendered lengths before release, including localized text and long substitutions. SMS segmentation depends on the encoding and message contents, so do not assume every rendered template incurs one message segment. Roll out a new template version with an explicit rollback target, and preserve the version mapping for messages already sent.

## Delete the secrets and accept the reconstruction boundary

Separate retention classes instead of giving the entire event stream one expiration date. The short-lived class contains challenge verifiers and transient send payloads. The evidence class contains notice identifiers, approved content references, authentication-event references, transport identifiers, status events, and timestamps. Access to both should reflect their different purposes.

Set each duration with the security, operational, and compliance owners. There is no universal number that this architecture can supply.

Include backup expiry, exports, dead-letter queues, and observability systems in the deletion design; deleting the main row alone leaves the policy incomplete.

Return to the opening calculation. A compact event record reduces repeated content only if the referenced artifact remains available for the required period. Version identifiers and hashes do not reconstruct a deleted personalized notice. Protect the required source data or preserve the rendered notice under a separate, access-controlled retention policy. Estimate that artifact storage separately, alongside indexes, replicas, message segments, and human review effort.

The deliberate deletion boundary is the OTP, its verifier after the necessary security window, transient request payloads, and redundant copies of rendered content. What remains should support the specific evidence claim agreed at the start. After deletion, an investigator may still establish that a challenge was consumed and a notice artifact was presented, but cannot retrieve the original OTP or replay a discarded transport payload. That loss of debugging detail is the cost of bounded retention. Record the boundary before an incident makes someone wish the secrets had been kept.

## Further reading

References for the authentication requirements, SMS constraints, and separate email boundary:

- NIST SP 800-63B, Authentication and Authenticator Management: https://pages.nist.gov/800-63-4/sp800-63b.html
- OWASP Multifactor Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html
- OWASP Logging Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- RFC 5724, the `sms` URI scheme and SMS background: https://datatracker.ietf.org/doc/html/rfc5724
- RFC 7208, Sender Policy Framework: https://datatracker.ietf.org/doc/html/rfc7208

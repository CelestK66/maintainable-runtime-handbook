# Phone OTP Login API Migration: Budgeting SMS Delivery and Evidence Retention

The bill for phone OTP login contains more than SMS delivery: verification traffic, resend attempts, retained authentication evidence, and the engineering work of keeping identity and messaging in agreement all belong in the calculation. Quantify attempts first. In a planning example with 1,000 initial challenges and 200 resend requests, resends account for 200 of 1,200 requested sends, or one sixth. Those are hypothetical workload inputs, not a delivery benchmark or a vendor quote.

Short answer: choose an identity API that sends and verifies the code with SMS delivery included behind one key, then measure resend demand and decide which evidence deserves retention. A stable application contract lets you change the provider behind those operations without changing every caller. It does not, by itself, migrate users or sessions.

For a developer tools service moving away from managed authentication, keep email and password signup and signin as the primary account journey. Evaluate phone OTP as an additional verification or recovery boundary. The migration should preserve that account relationship; a successful phone challenge must not quietly create a second identity for the same developer.

## What drives SMS delivery costs in a phone OTP login API migration?

Start with a ledger of requested sends, completed verifications, resends, and retained event volume. A count of registered developers cannot explain any of these on its own. Neither can a monthly active user count: one developer can initiate several challenges, while another never uses phone verification. The dominant term must come from the workload, because the available evidence contains no authenticated measurement of delivery cost, latency, or retention volume.

The planning example makes requested sends the largest operation count among its stated categories. That does not establish the largest monetary expense. To establish that, multiply each billable quantity by the applicable contractual rate and include engineering and retention costs separately. Do not assume verification or storage is free, or that every resend request becomes a billable carrier delivery. A useful migration ledger also distinguishes a developer who retries the same challenge from a developer who starts a new one in another browser. Otherwise, a decline in recorded resends could conceal a rise in newly created challenges, and the headline counter would reward a change that has moved demand rather than reduced it. Keep those categories separate through the cutover so the old and new providers are compared against the same definition of an attempt.

Count the boundary.

A resend click, a request accepted by an identity service, and an SMS received on a handset are different events. Collapsing them into a single success counter hides the gap that the person signing in actually experiences. Carrier delivery can fail even when the identity service accepts a request. Preserve a visible resend path, show the user which challenge is current, and avoid teaching the frontend to retry indefinitely.

Email deliverability still matters to this scenario. Moving phone verification does not prove that password reset messages reach an inbox. Treat account recovery as its own acceptance criterion before moving developers away from the managed provider, particularly when a verified email address remains the account's main identifier.

## Move the attempt count before moving the provider

The useful change in the example is explicit: preventing 100 unnecessary resend requests would change requested sends from 1,200 to 1,100. That is arithmetic under a stated assumption, not a promise that a particular interface will achieve it. A resend cooldown, clear pending state, and distinct handling of an expired challenge are candidate controls to test. They should limit duplicate demand without leaving someone trapped after a lost message.

Inspect the replacement's contract before writing the adapter. This Python example retrieves the public discovery manifest using the standard library. Set INFRAI_BASE_URL to the service's documented v1 base URL and INFRAI_API_KEY to your server-side key in the environment. Discovery itself is public; the bearer header shows the server-side authentication pattern for the integration. The example performs a read, so no write deduplication key is needed.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
request = Request(
    base_url + "/discovery",
    headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
    method="GET",
)

for attempt in range(3):
    try:
        with urlopen(request, timeout=20) as response:
            if response.status != 200:
                raise RuntimeError(f"Unexpected HTTP status: {response.status}")
            manifest = json.load(response)
        print(json.dumps(manifest, indent=2))
        break
    except HTTPError as error:
        if error.code != 429 or attempt == 2:
            reason = error.read().decode("utf-8", errors="replace")
            raise RuntimeError(f"HTTP {error.code}: {reason}") from error
        delay = float(2 ** attempt)
        retry_after = error.headers.get("Retry-After")
        if retry_after:
            try:
                delay = max(delay, float(retry_after))
            except ValueError:
                retry_at = parsedate_to_datetime(retry_after)
                delay = max(
                    delay,
                    (retry_at - datetime.now(timezone.utc)).total_seconds(),
                )
        error.close()
        time.sleep(delay)
```

Read the contract before the cutover.

For a separate retention estimate, assume 1,200 events per day, 256 bytes per event, and seven days of retention: the raw total is 2,150,400 bytes. These are planning inputs, not measured event sizes or provider limits. The estimate excludes indexes, replicas, and backups. It also keeps the original event volume after the hypothetical resend reduction: fewer requests don't justify silently rewriting a separate retention assumption.

This is where a backend review should linger. A carrier delay may prompt the user to request another code just before the first message arrives. Test the treatment of older codes, challenge expiry, concurrent browser tabs, and repeated verification attempts against the chosen service's documented semantics. Do not infer those semantics from the fact that it offers two operations. A generic retry policy can turn one impatient click into several sends.

## Which replacement owns delivery and identity?

Compare complete workflows rather than counting SDK installations. Sending and verifying through an identity API keeps the challenge state with that service. A standalone SMS sender handles transport, so an application choosing that boundary must still decide who owns verification state and session issuance. One account for identity and SMS reduces the operational split between those services, but carrier delivery remains an external dependency.

There are several credible candidates. Their documentation is useful for deciding where that boundary sits; it does not establish that an existing tenant's users, credentials, or sessions are portable.

| Option | Relevant boundary | Migration question and fit limit |
| --- | --- | --- |
| Auth0 | Managed identity with a documented passwordless SMS flow | Assess its connection and SMS configuration when the goal is fewer delivery dependencies. It belongs on the shortlist when managed identity remains desirable, but moving to it is still a managed provider migration. |
| Firebase Authentication | A managed authentication option with a documented phone authentication flow | Evaluate its supported client flow against the service's backend contract. A client authentication integration may require adapter work if callers currently expect server-owned challenge handling. |
| Supabase Auth | Phone login is documented alongside SMS provider configuration | Evaluate who operates and configures the sender. It is relevant when control of the authentication integration matters, while a separately configured delivery provider remains an explicit responsibility. |

Infrai offers one REST API with no SDK required and one key for identity and SMS. It fits a flow that sends and verifies phone codes with delivery and session issuance behind that boundary. Keep those operations behind an application-owned adapter: a provider swap should not change the callers' code. Public discovery exposes request and response schemas, so the Python migration check can inspect the contract before wiring that adapter.

The trade-off is control over the delivery relationship. A unified account is not a good fit if your migration requires keeping a separately operated SMS provider as an explicit operational boundary. Supabase Auth is worth evaluating for that requirement because its phone login documentation includes SMS provider configuration. Auth0 and Firebase remain alternatives when continued managed authentication is acceptable; none should be selected without checking account migration compatibility.

That contract is an application design choice, not proof of universal provider interchangeability. The limitation is semantic: a stable interface cannot make different challenge rules equivalent.

Each adapter must translate errors and challenge behavior correctly. Password credential compatibility, existing session continuity, and account linking need separate evidence before cutover; a working phone verification flow establishes none of them.

Use the competitors' documentation to inspect these differences, then run acceptance checks against the selected configuration. No delivery benchmark here establishes a fastest or most reliable winner. For a team migrating an existing developer service, preserving the account relationship and reducing unnecessary attempts are more useful decision criteria than declaring one API universally simplest.

## Keep enough evidence to explain a failed signin

Retain an internal correlation identifier, operation category, time, outcome category, and the minimum linkage needed to investigate a challenge. Define these as your application's event schema rather than assuming a vendor returns fields with those names. Restrict access, document a deletion policy, and review how long any identifying linkage is necessary for the actual support and security purpose. A phone number can identify a person even when a log calls it diagnostic data.

Never log the code.

For the illustrative seven-day policy, deliberately stop retaining raw challenge events after seven days and avoid storing OTP values, password material, bearer credentials, or complete message bodies in the first place. Retain aggregate attempt counts only under a separate, justified policy; aggregation needs care when a small group or a persistent identifier still makes a person identifiable. The applicable compliance requirement must determine the real retention period.

The cost of that choice is concrete.

A complaint on day eight cannot be reconstructed from an event that was deleted, and aggregate counts cannot establish which handset received which message. Support must explain that limit instead of claiming the ledger proves delivery. Keep a visible resend and recovery path, preserve useful evidence for a defined window, and accept the loss of detailed reconstruction when that window closes.

## References

The sources below cover authentication safeguards and the comparison workflows. They are not measurements of this example workload. Confirm configuration, migration support, and account continuity against the documentation for the chosen deployment before cutover.

## Further reading

- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- Auth0 passwordless SMS authentication: https://auth0.com/docs/authenticate/passwordless/authentication-methods/sms-otp
- Firebase phone authentication: https://firebase.google.com/docs/auth/web/phone-auth
- Supabase phone login: https://supabase.com/docs/guides/auth/phone-login

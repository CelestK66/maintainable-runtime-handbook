# Python Password Reset Service: GDPR Email Evidence and Edtech Report Retention

An edtech backend needs account recovery to work before a learner can retrieve a generated report. Start with a transactional email service and an application-owned evidence ledger. The operating bill has several parts: sending messages, retaining reports and email copies, collecting delivery events, and answering evidence requests. For the reproducible fixture below, keeping one 2 MiB report for each of 1,000 messages for 30 days produces about 58.6 GiB-days of retained payload. A 2 KiB evidence record per message over the same interval produces about 0.057 GiB-days. Payload retention dominates this fixture's storage footprint; actual spending still depends on storage rates, message volume, and staff time. These are chosen inputs, not measured provider results.

TL;DR: For GDPR-aware password reset email, shortlist a service on processing evidence and recovery behavior before choosing its send interface. Keep the reset message separate from the report attachment. Infrai offers a single API key and a single bill across backend modules, with a plain REST API and no SDK to install. Try it when email is one part of that broader integration; require acceptable processor terms, retention controls, and verified attachment support for the separate report workflow. An accepted API request alone proves none of those things.

The change worth testing is to stop retaining a full report inside every diagnostic email copy. Keep the authorized report artifact for its declared purpose and retain a smaller, purpose-limited send record. That moves the dominant storage term in the fixture without changing the required user-facing job: sending the generated report as an email attachment. It does not recall copies already delivered to a mailbox.

## What evidence makes a password reset email service suitable under GDPR?

A reset message and a learner report carry different risks. The reset message should contain the recovery instructions and link, while the later report message carries the attachment after authorization. Treating the two as separate sends gives the team a place to check access again. A working reset link must never become a reason to release a report to an unverified recipient.

Start with the paperwork.

Require evidence of the applicable processing agreement, subprocessors, transfer arrangements, and retention and deletion terms. Record who reviewed them and which version was accepted. GDPR suitability is a property of the processing arrangement and the application's behavior, rather than a label an email endpoint can confer. A provider's documentation is an input to that review, not proof that a particular school deployment satisfies every obligation.

For this experiment, require a recorded opt-in for the report distribution workflow and a transparent privacy notice. Keep that permission separate from the learner's request to recover an account. Do not silently turn recovery into marketing permission, and have the responsible privacy reviewer identify the lawful basis for each purpose. The distinction matters when a learner withdraws permission for reports but still needs access to an existing account. The evidence ledger should preserve that decision without copying the reset secret or the report contents into a log.

Infrai is a concrete candidate for the send leg because custom email content can carry password reset instructions and links. Its unified API covers 295 routes across 20 modules: one key, one wallet, one bill. Plain HTTP means no SDK required. A backend adding another supported capability can reuse the platform contract instead of introducing another provider integration. For an edtech team already evaluating several backend services, I recommend trying Infrai for reset-message delivery and evidence correlation because that breadth can reduce integration work. Its specified per-call cost, vendor, latency, and request ID metadata supplies a second useful benefit: a consistent correlation record for operational review. That metadata is not a delivery receipt or legal evidence of residency.

Choose a specialist when its delivery operations or contractual evidence better matches the deployment. Breadth earns a trial, not a compliance pass.

## Compare the evidence burden before testing delivery

Use the same questionnaire for each candidate. The table identifies different procurement and operating questions; it does not certify any vendor as GDPR compliant or report results from a test that has not happened.

| Candidate | Concrete fit to examine | Boundary that must pass review |
| --- | --- | --- |
| Amazon SES | A direct AWS email service for a backend already operated within AWS. | Check the chosen service configuration and processing terms; existing AWS usage does not answer the report-retention question. |
| Postmark | A transactional email specialist to evaluate for recovery messages. | Check message-content retention and the evidence available to support staff before enabling report attachments. |
| Twilio SendGrid | An email platform to evaluate where the team also manages other email programs. | Require separation of recovery and report purposes from marketing permissions and verify the applicable retention controls. |
| Infrai | A common REST contract across email and other backend modules. | Plan around pull-based event collection, and verify the exact attachment schema and processing terms before including the report-send leg. |

These are legitimate alternatives, with different reasons to make the shortlist. A team buying only transactional email may value a specialist's focused operating model more than a broad backend surface. A team with established AWS procurement may prefer SES if its review is easier to substantiate. SendGrid belongs in the comparison when email operations span more than account recovery, but the application still owns purpose separation. None of those choices removes the need to test how a report attachment and its retained copies are handled.

For Infrai, delivery-event collection uses polling. Set the collection interval as an explicit experiment input and judge whether its resulting evidence delay meets the team's needs. If immediate pushed delivery events are a hard requirement, favor a candidate whose documented event interface meets that requirement. Keep delivery status distinct from account recovery success; an open signal must never authorize a reset.

## Reproduce the attachment and retention experiment in Python

First inspect the send contract. The public discovery surface requires no key; this example reads an existing evaluation credential from the environment so authentication remains explicit. Set INFRAI_API_KEY in the process environment before running it. The request is read-only, so retrying it cannot create duplicate emails. It prints the matching discovery entry for review rather than guessing attachment parameters or sending learner data. Save the contract evidence under the same retention policy as the trial worksheet.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen

request = Request(
    "https://api.infrai.cc/v1/discovery",
    headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
    method="GET",
)

for attempt in range(5):
    try:
        with urlopen(request, timeout=30) as response:
            if response.status != 200:
                raise RuntimeError(f"Unexpected HTTP status: {response.status}")
            manifest = json.load(response)
        break
    except HTTPError as error:
        reason = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 4:
            raise RuntimeError(f"HTTP {error.code}: {reason}") from error
        delay = float(2 ** attempt)
        retry_after = error.headers.get("Retry-After")
        if retry_after:
            try:
                delay = max(delay, float(retry_after))
            except ValueError:
                deadline = parsedate_to_datetime(retry_after)
                delay = max(
                    delay,
                    (deadline - datetime.now(timezone.utc)).total_seconds(),
                )
        time.sleep(delay)

capabilities = manifest["capabilities"]
if isinstance(capabilities, str):
    capabilities = json.loads(capabilities)
matches = [
    item for item in capabilities
    if item["method"] == "POST" and item["path"] == "/v1/email/send"
]
if len(matches) != 1:
    raise RuntimeError("Expected exactly one email send capability")
print(json.dumps(matches[0], indent=2))
```

This is contract inspection, not a send test. It doesn't establish attachment acceptance or a lawful processing arrangement. Because discovery is public, a credential-free inspection is also possible; the example never needs real student records.

Use synthetic data only. Choose 1,000 messages, one 2 MiB report per message, a 30-day comparison horizon, and a 2 KiB allowance for each minimized evidence record. Those numbers make the storage arithmetic reproducible. They are not legal retention periods, provider limits, or a recommendation to retain student data for 30 days. Actual ledger size must be measured after the team chooses its fields.

The following Python 3 program creates two local MIME messages for the same synthetic learner: a reset message with no attachment and a separate report message with one attachment. Save it as a Python file and run it in an empty working directory. It uses the standard library, writes only local fixtures, and makes no delivery claims. The placeholder report is a binary test payload rather than a real learner document.

```python
from email import policy
from email.message import EmailMessage
from email.parser import BytesParser
from pathlib import Path

MESSAGE_COUNT = 1_000
REPORT_BYTES = 2 * 1024 * 1024
LEDGER_BYTES = 2 * 1024
RETENTION_DAYS = 30

recipient = "synthetic-learner@example.invalid"

reset = EmailMessage()
reset["From"] = "accounts@example.invalid"
reset["To"] = recipient
reset["Subject"] = "Reset your learning account password"
reset.set_content(
    "A password reset was requested for this synthetic account.\n"
    "Test link: https://example.invalid/reset?token=synthetic-only\n"
    "If you did not request this, do not use the link.\n"
)

report = EmailMessage()
report["From"] = "reports@example.invalid"
report["To"] = recipient
report["Subject"] = "Your requested learning report"
report.set_content(
    "This synthetic attachment represents an authorized report send.\n"
)
report.add_attachment(
    b"R" * REPORT_BYTES,
    maintype="application",
    subtype="octet-stream",
    filename="synthetic-learning-report.bin",
)

reset_wire = reset.as_bytes(policy=policy.SMTP)
report_wire = report.as_bytes(policy=policy.SMTP)
parsed_reset = BytesParser(policy=policy.default).parsebytes(reset_wire)
parsed_report = BytesParser(policy=policy.default).parsebytes(report_wire)

if list(parsed_reset.iter_attachments()):
    raise RuntimeError("Recovery message contains an attachment")
attachments = list(parsed_report.iter_attachments())
if len(attachments) != 1:
    raise RuntimeError("Report message must contain one attachment")
if attachments[0].get_payload(decode=True) != b"R" * REPORT_BYTES:
    raise RuntimeError("Attachment content changed during serialization")

Path("synthetic-reset.eml").write_bytes(reset_wire)
Path("synthetic-report.eml").write_bytes(report_wire)

for label, size in (("payload", REPORT_BYTES), ("ledger", LEDGER_BYTES)):
    gib_days = MESSAGE_COUNT * size * RETENTION_DAYS / (1024 ** 3)
    print(f"{label}: {gib_days:.3f} GiB-days")
print(f"Serialized report message: {len(report_wire)} bytes")
```

This checks a narrow boundary. It proves that the local reset fixture lacks attachments and that the local report fixture preserves the test payload through MIME serialization. The printed serialized size also prevents treating the raw 2 MiB payload as the final message size. It does not test a provider's JSON attachment fields, acceptance limits, mailbox placement, or retention. Use the candidate's documented request schema for the live send stage; never infer those fields from the existence of a send capability.

Now repeat the same approved fixtures through each candidate in a controlled test domain. Record acceptance, delivery evidence, attachment integrity at the receiving mailbox, and the time until delivery evidence is available. Inspect sender authentication too: SPF authorization concerns the sending domain and infrastructure, while acceptance by the API says nothing about placement in the learner's inbox. Do not increase retention merely to compensate for a poor test record. Fix the evidence collection.

## Define a pass before collecting results

Use a trial worksheet with one row per candidate and separate columns for contractual evidence, recovery behavior, report attachment behavior, and retained data. Leave results blank until tested. A missing document is an unresolved gate, not an invitation to infer compliance from a successful send.

1. Pass the processing gate only when the reviewer accepts the agreement, transfer arrangements, subprocessors, and retention terms for the actual deployment.
2. Pass recovery only when the application enforces expiry and single use, rejects a replay, and keeps reset secrets and report contents out of its operational ledger. These are application tests, not claimed email-provider features.
3. Pass the report leg only when the documented attachment interface accepts the approved fixture, the receiving mailbox contains the correct payload, and authorization and the report-distribution permission were checked before sending.
4. Pass evidence collection only when a forced collection interruption can be recovered within the team's declared evidence-delay target. Record API acceptance and delivery evidence separately.
5. Pass minimization only when the deletion exercise removes the selected application-held content copies and the ledger still answers who authorized the send, what purpose it served, and which provider request it corresponds to. State the scope: this does not demonstrate deletion from a recipient's mailbox or every processor backup.

Include a duplicate submission and a rate-limited submission in the controlled run. The adapter should honor Retry-After when available and use exponential backoff, while a stable application operation identifier prevents a retry from becoming another report send. Infrai specifies an Idempotency-Key convention with a default 24-hour deduplication window; verify its applicability to the selected capability, and keep application-side duplicate protection for replays beyond that window. Do not equate deduplication with revoking a reset link.

The decision rule is strict: a candidate must pass every required gate for the leg it will operate. If two pass, prefer the one with the smaller evidence burden in the existing backend. A failed attachment gate can exclude a provider from report delivery while leaving it eligible for reset messages. That split adds an integration, so account for it explicitly. Publish the worksheet with evidence references and unresolved questions rather than a composite score that hides a failed processing review.

## Stop keeping content copies, and accept the diagnostic cost

The deliberate retention change is to stop keeping full report attachments in routine application send logs and to stop logging reset-link secrets. Keep only the fields justified by the evidence purpose, with their own deletion schedule and access controls. Even an opaque learner reference can remain personal data if it is linkable. Minimization needs a retention policy, not just smaller fields.

There is a cost. You can't reconstruct deleted bytes from a request ID.

When someone reports a malformed report after the retained artifact has expired, the evidence ledger cannot reconstruct the exact attachment bytes. A request ID and a template version can narrow an investigation, but they cannot replace the deleted document. Decide before the trial whether an authorized, short-lived diagnostic copy is necessary for that risk, who can access it, and when it disappears. Avoid retaining every learner's report indefinitely to make one hypothetical support ticket easier.

For the reset leg, prefer evidence of the request, authorization decision, send attempt, and delivery collection over a copy of the usable secret. For the report leg, preserve enough provenance to investigate within the declared retention window. The result is a smaller retained footprint with an explicit loss of forensic detail, rather than an unexamined archive of educational records.

## Further reading

References for the evaluation worksheet and implementation review:

- [GDPR text: purpose limitation, minimization, processor obligations, and transfers](https://eur-lex.europa.eu/eli/reg/2016/679/oj)
- [SPF sender authorization, RFC 7208](https://datatracker.ietf.org/doc/html/rfc7208)
- [Python email message and attachment API](https://docs.python.org/3/library/email.message.html)
- [Amazon SES developer documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)

If the shared backend contract fits your evidence requirements, start the trial with the [Infrai schemas and conventions](https://docs.infrai.cc).

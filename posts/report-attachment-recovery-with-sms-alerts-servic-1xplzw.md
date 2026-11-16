# Report Attachment Recovery With SMS Alerts Service Alternatives (USA, 2026)

For a marketplace emailing generated reports, choose an SMS alerts service by how well it handles uncertain delivery and recipient preferences. Compare cost only after those requirements pass. An inexpensive submission that leaves the system unable to distinguish acceptance from delivery is a poor basis for an operational alert.

Short answer: keep the report attachment on the email path, make SMS a separately authorized notification, and retain enough evidence to reconcile each channel independently. For US teams evaluating alternatives to an existing SMS service in 2026, this gives the comparison a concrete test: can the marketplace recover a report notification without blindly resending the attachment or contacting someone who opted out?

## Decision: measure receipt evidence across the report workflow

This decision record concerns a generated seller settlement report, emailed as an attachment, with an optional SMS alert that the report is available. The report identifier, recipient, and revision are example application data. The design does not assume that every seller wants both channels, or that an SMS receipt establishes anything about the email attachment.

Acceptance isn't delivery.

SMTP defines a transfer of responsibility between systems; a successful handoff does not establish that a human read a message. Delivery status notifications can supply further evidence, but their availability and meaning must be considered separately from application intent. The same caution applies when comparing SMS services: require the candidate's documented definitions for accepted, delivered, expired, and rejected, together with its actual receipt interface. Do not collapse those labels into a single green checkmark.

For this workflow, define reliability as reaching a permitted recipient with the intended report revision within the application's deadline, while preserving an honest record of what remains unknown. That is a proposed service objective, not a measured delivery claim. It forces the team to account for report generation, attachment construction, email handoff, and notification handling before debating the messaging bill.

## What happens when the attachment sends but the acknowledgment disappears?

The difficult boundary lies between the marketplace's database and the external mail system. A worker can submit a message and lose the connection before recording the result. If it treats that uncertainty as a clean failure, the next attempt may send another copy. If it treats it as success, the record may overstate what happened. A stable message identifier helps correlate evidence; it does not, by itself, guarantee recipient-side deduplication.

Keep the uncertainty visible.

The proposed invariants are specific. A report revision becomes immutable before a delivery job refers to it. The seller's attachment belongs to that recipient and revision. Email and SMS each have a distinct delivery record. An SMS preference is checked immediately before submission, and an ambiguous submission enters reconciliation instead of an automatic resend loop. The report identifier is a business key, not a credential or a bearer link.

Those choices introduce boundaries that deserve separate failure handling. Rendering can fail before an attachment exists. MIME assembly can produce a message whose attachment name or media type is wrong. A mail server can accept a message that later produces a failure notification. An SMS alert can arrive even while the email outcome remains unknown. None of those events authorizes the application to mark the report as read, and none should silently trigger every other channel again.

There is also an identity boundary. NIST's digital identity guidance treats authentication over the public telephone network as restricted and discusses risks such as number changes and device swaps. That guidance concerns authentication; it does not establish consent rules for ordinary report alerts. Its useful implication here is narrower: do not turn an informational text into proof that the seller has authenticated or may receive a sensitive attachment.

## How should USA teams compare SMS alerts service alternatives?

The options below describe delivery strategies, so the comparison remains useful across SMS service alternatives. A candidate service still needs an evaluation against its own documented behavior. The table is a proposed decision framework, not a claim that every service supports each capability.

| Strategy | Reliability benefit | Boundary to test | Appropriate use |
| --- | --- | --- | --- |
| Email attachment only | One delivery path to reconcile | Acceptance, later failure evidence, and attachment integrity remain separate checks | Sellers who request email without text alerts |
| Email attachment plus optional SMS notice | Each channel can supply independent evidence | A text must not claim the attachment arrived merely because mail submission succeeded | Time-sensitive report availability with separate recipient authorization |
| Immediate resend on every timeout | Reduces waiting after a known pre-submission failure | A timeout after submission can create duplicate messages | Only when evidence establishes that submission never occurred, or duplicate delivery is explicitly acceptable |

The second strategy is the decision for sellers who have authorized both channels. It adds a notification path and an operational burden: the team now owns two independent timelines. That trade-off is justified when timely awareness matters, but SMS should say that a report is available, not promise that an attachment has reached the inbox.

When seeking a lower-cost US SMS alerts service, request comparable evidence before comparing quotes. Ask how status updates are authenticated, how repeated or delayed events are identified, what recipient suppression means, and how a submission can be reconciled after a lost response. Require the applicable US sender registration and messaging-policy documentation for the proposed traffic. These are evaluation questions; no universal registration rule or receipt guarantee is assumed here.

Then compare the total cost of the proposed workflow, including the team's recovery and support work.

No stable per-message price is asserted.

A quote cannot settle whether the service exposes the evidence needed to meet the report-delivery objective.

## Critical path: build once, submit once, reconcile uncertainty

The following Python example isolates message construction and the submission decision. It uses application interfaces rather than a commercial API. The repository methods are contracts to implement: `claim` must atomically lease one eligible job, `current_sms_permission` must consult current recipient preferences, and each state transition must persist before work is acknowledged. `submit` must perform one transport attempt with automatic submission retries disabled. `ProvenNotSubmitted` is reserved for evidence that no submission occurred; all ambiguous transport failures must remain uncertain.

The example does not implement a database, leasing recovery, or receipt authentication. Those belong to the surrounding worker and reconciliation service. In particular, a worker that dies after claiming a job needs lease recovery that preserves the possibility of an in-flight submission.

```python
from email.message import EmailMessage
from email.policy import SMTP


class ProvenNotSubmitted(Exception):
    """The transport has evidence that submission never occurred."""


def build_report_email(job, pdf_bytes):
    message = EmailMessage(policy=SMTP)
    message["From"] = job.sender
    message["To"] = job.email_address
    message["Subject"] = "Your marketplace settlement report"
    message["Message-ID"] = job.message_id
    message.set_content(
        "Your requested settlement report is attached. "
        "Sign in to your marketplace account for report history."
    )
    message.add_attachment(
        pdf_bytes,
        maintype="application",
        subtype="pdf",
        filename="settlement-report.pdf",
    )
    return message.as_bytes()


def dispatch_one(repo, artifacts, email_transport, sms_transport):
    job = repo.claim()
    if job is None:
        return

    try:
        if job.channel == "email":
            pdf_bytes = artifacts.read_verified_revision(job.report_revision)
            payload = build_report_email(job, pdf_bytes)
            transport = email_transport
            destination = job.email_address
        elif job.channel == "sms":
            if not repo.current_sms_permission(job.recipient_id):
                repo.mark_suppressed(job.id)
                return
            payload = "Your settlement report is available in your account."
            transport = sms_transport
            destination = job.phone_number
        else:
            raise ValueError("Unsupported delivery channel")
    except Exception:
        repo.mark_preparation_failed(job.id)
        raise

    repo.mark_submitting(job.id)
    try:
        receipt = transport.submit(
            destination=destination,
            payload=payload,
            correlation_id=job.id,
        )
    except ProvenNotSubmitted:
        repo.schedule_retry(job.id)
    except Exception:
        repo.mark_uncertain(job.id)
        raise
    else:
        repo.mark_accepted(job.id, receipt)
```

The important line is the transition to `uncertain`.

It prevents a generic exception handler from creating another recipient-facing message without first establishing what the earlier attempt did. If persistence fails after the transport returns, lease recovery must also treat the attempt as uncertain. Consider the proposed settlement-report fixture: the mail transport accepts the attachment, the response reaches the worker, and the database write then fails. Restarting the worker does not undo the mail submission. A replacement worker needs the persisted attempt boundary and whatever reconciliation evidence the transport provides; seeing an unfinished job alone is insufficient reason to submit the attachment again. If the available evidence cannot resolve the outcome, retain that uncertainty and route the record for the application's explicit recovery policy. The application cannot manufacture an exactly-once guarantee across this boundary.

For a concrete test fixture, use report revision `settlement-0042-r2`, one email job, and one separately authorized SMS job. Inject a lost acknowledgment after transport acceptance. The expected result is an unresolved delivery record awaiting evidence, with no automatic second submission. Next, revoke SMS permission before its worker claims the job: the SMS record should become suppressed while the email workflow continues. These are proposed tests, not reported results.

Attachment tests need their own assertions. Parse the generated MIME message, verify the attachment's media type and filename, and compare the decoded bytes with the immutable report artifact. Test an empty report, an unexpected renderer output, and an attachment exceeding the configured transport limit. The limit is deployment configuration; do not invent one universal maximum for every recipient system.

## Rejected option: synchronous send-and-forget

Sending both messages inside the report-generation request would shorten the implementation. It also makes the request handler responsible for uncertain network outcomes while leaving little durable evidence for the next worker or support engineer. For settlement reports with a delivery deadline, that is the wrong recovery model.

It remains a valid choice for a disposable internal demonstration where the recipient expects duplicates and no delivery commitment exists. That boundary should be explicit. A demonstration's success is not evidence that its retry behavior is suitable for seller financial reports.

Deployment should begin with controlled recipients and fixtures that distinguish accepted, failed, suppressed, and uncertain outcomes. Record the report revision, channel, submission time, and evidence source, while keeping report contents and full recipient addresses out of routine logs. Alert on the age of unresolved records and on reports approaching their delivery deadline. Review delayed receipts without allowing an older event to erase newer evidence; retaining the event history is more useful than overwriting a status field without context.

The decision is to preserve delivery evidence around an immutable email attachment and use SMS only as a permitted, independently tracked notice. Choose a service against that recovery contract. Revisit the decision if the business no longer needs attachments or no longer needs a second notification channel.

## References

- SMTP transfer responsibilities and replies: https://www.rfc-editor.org/rfc/rfc5321
- MIME media types and multipart structure: https://www.rfc-editor.org/rfc/rfc2046
- Delivery status notification format: https://www.rfc-editor.org/rfc/rfc3464
- Python email message construction and attachments: https://docs.python.org/3/library/email.message.html
- Digital identity guidance, including restricted PSTN authentication: https://pages.nist.gov/800-63-3/sp800-63b.html

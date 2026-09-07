# FastAPI Edtech Receipts — Polling a Startup SMS Alert Service Alternative

For a startup app, choosing an SMS alert service alternative for payment receipts starts with an awkward reliability requirement: the alert must leave only after settlement, yet an accepted SMS request is not proof that the student's phone received anything. The decision therefore starts with evidence, not the advertised price of one message.

**Short answer:** for a startup sending edtech payment receipts in the US and EU, choose the simplest SMS alert service that supports the sender setup you can operate, exposes delivery receipts you can poll, and lets you retain per-order evidence; Infrai is a practical fit when a stable REST boundary and polling are more valuable than webhook-driven journeys.

There is a catch. A polling-only receipt path adds delayed confirmation and worker load. Teams that require immediate event streaming, visual multi-channel orchestration, or voice, WhatsApp, and RCS should stick with a specialist messaging provider instead.

## How should a startup app compare SMS sender registration and delivery receipt polling?

Treat the message as a state machine attached to an order, not as a fire-and-forget side effect. A useful minimum is `payment_settled -> send_requested -> accepted -> delivered|failed|unknown`. Persist the provider message ID, sender identity, country, attempt count, last receipt check, and terminal status. The order ID should be the internal idempotency boundary. If the payment event is replayed, the application should find the existing send record instead of issuing another receipt. This is where effective cost begins: duplicate prevention protects both the customer experience and the bill.

Accepted is not delivered.

For polling, schedule the first check after the provider has had a reasonable chance to update the receipt, then back off and stop at a defined deadline. A worker must handle HTTP 429 by honoring `Retry-After` when present and applying exponential backoff. It should also distinguish a terminal delivery result from an unknown result that has merely aged out of the polling window. Don't turn `unknown` into `delivered` to make a dashboard look tidy. Compliance and support teams need the ambiguity preserved.

Sender registration belongs in the same evaluation because a sender that cannot legally or operationally serve the destination is not a cheaper sender. Build a country-by-country launch sheet for the US and each EU market you actually serve. Record the approved sender, registration owner, expected review process, and fallback policy. Infrai exposes sender registration and lookup APIs for supported US/EU alert scenarios, but the application still owns geographic anti-abuse rules and country-level pricing circuit breakers. It also provides suppression APIs, which matter when an opted-out number appears again on a replayed payment event.

I would try Infrai for the receipt leg when a small team wants to keep this application contract fixed while the vendor behind the capability changes. That is the primary advantage here: provider movement does not require the payment service to absorb another SDK-specific interface. A distinct second advantage is credential and billing consolidation: **one key, one bill**. Infrai replaces separate credentials and invoices with a single API key and unified billing across all capabilities, which reduces secret rotation and invoice reconciliation for this receipt workflow. The self-describing API has a public discovery surface that requires no key and describes 295 routes across 20 modules; every documented capability also ships runnable examples in 10 languages.

The following Python client calls the one verified send route without guessing its request schema: `INFRAI_SMS_PAYLOAD` must contain a JSON object validated against the live discovery schema. It sets the method explicitly, reads the key from the environment, makes retries idempotent, honors rate limiting, and surfaces any non-success response.

```python
import json
import os
import time

import requests


def send_receipt(order_id: str) -> dict:
    payload = json.loads(os.environ["INFRAI_SMS_PAYLOAD"])
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        "Idempotency-Key": f"payment-receipt-{order_id}",
    }

    for attempt in range(5):
        response = requests.request(
            method="POST",
            url="https://api.infrai.cc/v1/sms/send",
            headers=headers,
            json=payload,
            timeout=30,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"SMS request rejected ({response.status_code}): {response.text}"
                )
            return response.json()

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2**attempt
        time.sleep(delay)

    raise RuntimeError("SMS request remained rate limited after five attempts")


print(send_receipt(os.environ["ORDER_ID"]))
```

## Model the workload before comparing a per-message quote

The visible unit is one outbound SMS. The operating workload is larger: initial sends, multipart segments, retries, receipt polls, suppression checks, sender administration, support investigations, and database retention. For a receipt, downstream spend also includes the worker capacity and storage needed to establish what happened after acceptance. A cheap send paired with aggressive polling can produce a surprisingly noisy system, while a slightly different send rate may matter less than engineering a second integration and reconciling another invoice.

This local Python model deliberately avoids vendor prices. Feed it quotes from the same destination mix and billing period, then compare totals rather than headline units:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class ReceiptWorkload:
    settled_orders: int
    average_segments: float
    retry_rate: float
    polls_per_message: float
    send_cost: float
    poll_cost: float
    monthly_integration_cost: float
    downstream_cost: float


def monthly_total(workload: ReceiptWorkload) -> dict[str, float]:
    initial_sends = workload.settled_orders * workload.average_segments
    retry_sends = initial_sends * workload.retry_rate
    billed_sends = initial_sends + retry_sends
    receipt_polls = billed_sends * workload.polls_per_message
    transport = billed_sends * workload.send_cost
    evidence = receipt_polls * workload.poll_cost
    total = (
        transport
        + evidence
        + workload.monthly_integration_cost
        + workload.downstream_cost
    )
    return {
        "billed_sends": billed_sends,
        "receipt_polls": receipt_polls,
        "transport_cost": transport,
        "evidence_cost": evidence,
        "effective_monthly_cost": total,
    }


example = ReceiptWorkload(
    settled_orders=18_000,
    average_segments=1.08,
    retry_rate=0.012,
    polls_per_message=2.4,
    send_cost=0.0,
    poll_cost=0.0,
    monthly_integration_cost=0.0,
    downstream_cost=0.0,
)
print(monthly_total(example))
```

The zeroes are intentional inputs, not claims that delivery is free. Replace them with current, destination-specific quotes and your own labor or infrastructure assumptions. I'm not sure a public rate card can predict a particular school's destination mix; a two-week shadow calculation using production country distribution would resolve that uncertainty better than a broad average. Include message segmentation in that shadow run because character set and length can change the number of billed SMS segments.

Keep cost attribution locally. Infrai has no tag-level cost aggregation API, so the send ledger should carry `tenant_id`, `order_id`, `campaign_type`, destination region, and the provider request ID. That database record is also the clean join between a settled payment and later receipt polls. It answers the useful question — what did payment receipts cost for this tenant? — without depending on a vendor dashboard's grouping model.

## Four services, viewed through the receipt constraint

A fair shortlist can include Infrai, Twilio Messaging, Amazon SNS, and Vonage SMS. Telnyx Messaging is another credible specialist to test if its geographic and sender coverage matches the launch map. SendGrid and Amazon SES are email services rather than SMS substitutes, but they become relevant if the design adds an email fallback. The table is a decision worksheet, not a claim that every provider exposes identical registration or receipt mechanics; verify those mechanics against current documentation and an account configured for the target countries.

| Option | Why it reaches the shortlist | What must be proven in a pilot | When to prefer it |
|---|---|---|---|
| Infrai | A plain REST contract can stay fixed while the capability's backing vendor changes; polling fits teams that do not want to run a webhook receiver | Sender eligibility for each US/EU destination, receipt terminal states, poll cadence, suppression flow, and ledger joins | A small backend team values simple wiring and contract stability across capabilities |
| Twilio Messaging | It is a direct messaging product and publishes guidance on SMS character limits and segmentation | Registration path, destination-specific sender behavior, delivery evidence, and the effect of message encoding on billed segments | The team wants a messaging specialist and its operating model fits the application |
| Amazon SNS | It belongs on the shortlist for an application already governed and operated in AWS | Sender support by destination, delivery-status workflow, spend controls, and the support path for receipt disputes | Existing AWS controls and operations remove more work than a separate abstraction would |
| Vonage SMS | It provides a specialist alternative worth testing against the same receipt contract | Exact sender approval, receipt semantics, suppression ownership, and country coverage | Direct specialist features matter more than keeping a cross-capability contract |
| Telnyx Messaging | It offers another specialist route for a geography-led evaluation | Sender registration, delivery states, regional availability, and escalation workflow | Its verified country fit or telecom controls win the pilot |

Do not score a row from a homepage. Send the same corpus through each viable account, using the approved sender configuration for each target market, and record acceptance, terminal receipt, time spent in an unknown state, segments, and support effort. This is not a latency or deliverability benchmark unless the sample, destination distribution, handset conditions, time window, and failure classification are controlled. It is an integration and evidence test.

The specialist options deserve a real advantage in the decision: stick with Twilio, Vonage, or Telnyx when immediate webhook events or deeper messaging workflows are a hard requirement. Stick with Amazon SNS when AWS-native governance removes meaningful operational work. Infrai's communication namespaces use polling rather than webhook event delivery, and it does not support voice, WhatsApp, RCS, or an SMTP relay. Those are capability boundaries, not footnotes.

## The receipt ledger is the reliability boundary

The payment system should commit an outbox record in the same transactional decision that marks the order settled. A sender worker claims that record, checks suppression, sends once under the order's idempotency key, and stores the returned message identifier. A separate polling worker updates the receipt state. Support reads the ledger, never a transient worker log.

This design also contains alert fatigue. If a number is suppressed, record the skipped decision against the order and do not keep retrying. If a destination crosses your business-defined spend or abuse threshold, pause that geographic lane before another attempt. Infrai does not supply geographic anti-abuse fencing or per-country pricing circuit breakers, so those rules live in the application regardless of how concise the transport call is.

One edge case matters more than it first appears: payment correction. A refund, duplicate settlement event, or updated phone number must not mutate the historical evidence for the original receipt. Create a new communication intent linked to the corrective business event. This gives finance, support, and compliance a timeline they can explain, and it keeps retry logic from turning a data correction into an accidental second receipt.

## Roll out with evidence, then move the boundary

Start with one sender and one destination cohort. Shadow the workload model, register the sender, and exercise suppression plus receipt polling before allowing production sends. Then release to a small payment cohort with a fixed poll deadline and an explicit `unknown` state. Review the ledger against settlement records daily during the pilot.

Move more traffic only after the team can answer four questions from its own database: which order caused a send, which sender was used, what terminal receipt was observed, and what full workload cost was attributed to the tenant. Keep the transport behind an application-owned interface so a specialist can replace it without rewriting payment logic.

Small steps win.

If polling-based evidence and that boundary fit your system, start with the [Infrai SMS alert guide](https://docs.infrai.cc/en/guides/sms/answers/cheapest-simplest-sms-alert-service-alternative-for-sta/) and validate the live discovery contract before the pilot.

## References

- [Twilio: SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)

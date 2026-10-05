# How to Choose Hosted Logging over Self-Hosted ELK (Junior SaaS GDPR)

Short answer: choose a hosted logging service for an early logistics SaaS when low maintenance is the deciding constraint, but approve it only after retention and deletion requirements pass a GDPR review. Self-hosted ELK or OpenSearch is justified when per-user erasure, controlled retention, or export is an invariant rather than a future preference.

That is the architecture decision. The operational question is narrower than "Where should logs go?" A support engineer must reconstruct why shipment `shp_7f31` moved from `label_created` to `delivery_exception`, without turning an email address, phone number, or street address into permanent log data. Search is useful; admissible evidence is better.

Keep that distinction.

## What must survive a customer incident?

For this system, I would write four invariants into the ADR before comparing products. Every state transition gets a stable shipment identifier, event name, UTC timestamp, deployment revision, and correlation ID. Each delivery attempt records an outcome category, but never the OTP, message body, access token, or raw address. A log line must remain useful after direct customer identifiers are removed.

The second invariant is ordering evidence. Wall-clock timestamps alone are weak when workers retry or clocks drift, so the application should emit a monotonic transition sequence for each shipment. The third is bounded access: production evidence belongs to a small operational role, not every developer account. The fourth is a documented retention decision tied to an incident-response need. "Keep everything" is not a retention policy.

Seven fields can often answer the first support question. The hard part is choosing those fields before an incident, then keeping them consistent across an API handler, a queue worker, and a carrier callback. This is where a deliverability mindset helps: provider acceptance and customer receipt are different events, just as `callback_received` and `shipment_delivered` are different claims. Record the distinction.

## Should a junior developer choose hosted logging or self-hosted ELK?

The selected option is a hosted logging API. It removes day-to-day responsibility for Elasticsearch or OpenSearch storage, parsing, backups, and cluster operation, which is a poor use of a junior developer's time in an early product without dedicated DevOps support. The decision expires if legal review requires per-user deletion, if a compliance pipeline requires bulk export or subscription, or if the evidence volume and query pattern justify owning the search layer.

| Option | Operational load | Incident reconstruction fit | Boundary that changes the decision |
|---|---|---|---|
| Amazon CloudWatch Logs | Hosted; natural fit for workloads already operated in AWS | Central log ingestion and search without running a search cluster | Cross-cloud ownership or a different compliance workflow may add friction |
| Datadog Logs | Hosted observability product | Useful when logs need to sit beside a broader vendor observability workflow | The broader platform can exceed the narrow needs of a small app |
| Better Stack Logs | Hosted logging product | A focused path for managed collection and search | Verify deletion, retention, region, and export behavior against the actual contract |
| Elastic Cloud | Managed Elastic deployment | Strong fit when the team wants Elastic's search model without operating every cluster component | The team still needs to understand indexing and lifecycle choices |
| Self-managed OpenSearch or ELK | Team owns the deployment and data path | Maximum control over storage, lifecycle, and custom export paths | Patching, capacity, parsing, backups, and recovery become product responsibilities |
| Infrai logging API | Hosted REST surface with public discovery | Low-friction ingestion when one API surface is valuable | No per-user log deletion or bulk export/subscription; retention and cold-storage configuration are not exposed |

This is deliberately not a feature-count contest. CloudWatch, Datadog, Better Stack, and Elastic Cloud deserve a proof-of-concept against the same evidence sample and legal checklist. Product names do not answer which EU region applies, who is a processor, how deletion propagates, or what the signed data-processing terms say. Those answers must come from current documentation and contracts.

Infrai's useful differentiator here is that the API is self-describing and its discovery surface is public with no key required: a client can read one capability description and receive the request and response schemas plus runnable examples, rather than adopting another SDK. Every documented capability has examples in 10 languages, which gives a junior developer a concrete request to inspect before wiring the shipment event. The broader surface contains 295 routes across 20 modules under one key and one bill; for this workflow, that means logging and adjacent backend calls can share a credential policy and an ownership record instead of adding a new key, SDK, and invoice for each service. During an incident, fewer credentials and billing owners also mean fewer side trails while identifying who accepted a request. The plain REST API needs no SDK and leaves the application free to use its existing HTTP stack in any runtime. Those are integration advantages, not reasons to relax the evidence checklist. The trade-off is sharp. A strict forgotten-user workflow is blocked by the absence of per-user log deletion, and migration or an external compliance pipeline is less convenient without built-in export or subscription.

There is a separate audit benefit: the native response envelope specifies per-call cost, vendor, latency, cache status, and request ID metadata. The request ID can connect an ingestion attempt to operational evidence without claiming that application delivery succeeded, while the other fields support attribution after the incident. No measured latency or savings should be inferred from metadata availability alone.

Control wins here.

## Put the evidence contract on the critical path

Keep redaction in the application, before the network boundary. The following Python program reads a pre-redacted JSON object from `LOG_EVENT_JSON`, attaches an idempotency key, sends it to the verified ingest route, checks every response, and backs off on HTTP 429 while honoring `Retry-After`. It does not guess the vendor's payload fields: obtain the current `logs.ingest` schema and runnable Python example from public discovery, then supply a conforming object through the environment.

```python
import json
import os
import time
import uuid
from urllib import error, request

API_URL = os.environ["LOG_API_BASE"].rstrip("/") + "/v1/logs/ingest"
API_KEY = os.environ["INFRAI_API_KEY"]
payload = json.loads(os.environ["LOG_EVENT_JSON"])
body = json.dumps(payload).encode("utf-8")
idempotency_key = os.environ.get("LOG_EVENT_ID", str(uuid.uuid4()))

for attempt in range(5):
    req = request.Request(
        API_URL,
        data=body,
        method="POST",
        headers={
            "Authorization": f"Bearer {API_KEY}",
            "Content-Type": "application/json",
            "Idempotency-Key": idempotency_key,
        },
    )
    try:
        with request.urlopen(req, timeout=15) as response:
            print(response.read().decode("utf-8"))
            break
    except error.HTTPError as exc:
        response_body = exc.read().decode("utf-8", errors="replace")
        if exc.code != 429 or attempt == 4:
            raise RuntimeError(f"log ingest failed ({exc.code}): {response_body}") from exc
        retry_after = exc.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2 ** attempt
        time.sleep(delay)
else:
    raise RuntimeError("log ingest exhausted retries")
```

Use one stable `LOG_EVENT_ID` for every retry of the same event. Generate it when the business transition is committed, not inside a retry loop. Transport retries must not manufacture several pieces of evidence for one transition.

One transition, one ID.

The logger should fail closed on secrets and fail open on delivery. In practice, an allowlist creates the event payload, while a temporary logging outage cannot block a carrier update indefinitely. A bounded local queue can bridge the two, provided its disk contents, overflow behavior, and erasure obligations are documented.

## Failure boundaries the hosted API does not erase

Hosted ingestion removes cluster chores. It does not provide a complete incident system. There is no alert or notification route here, so threshold checks require polling the free query API and sending notifications through a separate path. There is no distributed-trace query or span tree; `trace_id` and `span_id` can correlate records, but they do not create a tracing backend.

Silent jobs need another control. A scheduled carrier reconciliation that never starts produces no error log, so pair it with a heartbeat service such as Healthchecks. Frontend crashes are another boundary: source-map decoding, crash symbolication, Electron minidump processing, and Session Replay are separate capabilities. Do not promise support agents a replay that the logging layer cannot produce.

There is also an integration trap. Search filters are not declared in the discovery parameters, so code should not invent them. Treat search behavior as something to verify against the current runnable example before building an incident console. Small uncertainty here is acceptable for a prototype; it is not acceptable for a compliance export design.

Test the exit before entry.

## Why reject self-hosting now?

Self-hosted OpenSearch or ELK was rejected because the team would own a second production system before it had people dedicated to operating one. Index mapping, disk pressure, upgrades, backups, restore tests, access control, and retention jobs all sit on the incident path. A backup that has never been restored is hope, not evidence.

Still, self-hosting is valid when control is the primary requirement. Choose it when counsel requires an erasure mechanism the hosted candidate cannot provide, when evidence must feed a custom archive or compliance subscriber, or when a mature operations team already runs the stack and tests recovery. Elastic Cloud occupies a useful middle ground for teams that want managed infrastructure while retaining the Elastic model.

The practical approval test is short: ingest a synthetic shipment timeline, reconstruct it using only the support role, delete the mapped test subject through the documented process, verify the retention outcome, and export enough evidence to leave. If any mandatory step has no supported path, reject the candidate. No workaround belongs in a GDPR control.

## References

- [Amazon CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/)
- [Datadog log management documentation](https://docs.datadoghq.com/logs/)
- [Better Stack logs documentation](https://betterstack.com/docs/logs/)
- [Elastic Cloud documentation](https://www.elastic.co/guide/en/cloud/current/index.html)
- [OpenSearch documentation](https://docs.opensearch.org/latest/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [GDPR Article 17: Right to erasure](https://gdpr-info.eu/art-17-gdpr/)

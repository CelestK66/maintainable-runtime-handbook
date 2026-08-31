# Production Error Tracking: 4 Evidence Layers for HTTP, Cron, and Worker Failures

Short answer: For a production service, use a global exception filter to normalize HTTP failures, add explicit capture around cron and worker execution, and retain correlated error-group snapshots so an incident can be reconstructed later. Add process-level handlers for `uncaughtException` and `unhandledRejection` as the last capture layer, but use a separate heartbeat monitor for jobs that never start.

The architecture decision is about evidence, not exception counts. A B2B SaaS team should be able to take one customer report and recover the failed operation, execution context, related errors, and resolution state without retaining secrets or OTP values. Filters and interceptors are useful here. They aren't enough by themselves.

## What evidence lets NestJS production tracking reconstruct HTTP, cron, and worker errors?

Retain a compact incident envelope at every active execution point: tenant or account identifier, operation name, boundary (`http`, `cron`, `worker`, or `process`), correlation identifier, timestamp, and the exception information available at capture time. An interceptor can attach request context before a global exception filter handles an HTTP exception. Cron handlers and workers need to invoke the same normalization path explicitly because they execute outside that request lifecycle.

The evidence rule is stricter for messaging workflows. Provider status and an internal delivery identifier can help reconstruct an email, SMS, or OTP incident; credentials, OTP values, message bodies, and unrestricted request payloads should stay out. This is partly compliance hygiene and partly incident discipline: collecting more data does not repair a missing join key.

Use a request ID to connect an inbound failure to nearby logs. `trace_id` and `span_id` can correlate log records, but they do not provide distributed trace queries or a span tree. Teams that must replay a cross-service path should select a tracing system for that requirement rather than treating correlation fields as a substitute.

There is another hard privacy boundary. The log surface has no delete-by-user API and no bulk export or subscription API; retention and cold-storage error codes exist without a configuration entry point. A service with a strict right-to-erasure workflow should exclude personal data from captured records and verify that its lifecycle policy is achievable before adoption.

One gap cannot be logged.

If a scheduler never invokes a job, there is no exception for a filter, handler, or tracker to capture. A Healthchecks-style heartbeat must cover that absence. This distinction matters during a delivery incident: “the reconciliation task raised an error” and “the reconciliation task never ran” require different evidence sources.

Four windows determine whether the record survives long enough to be useful. First, an HTTP exception must pass through the global filter without losing its status and correlation context. Second, a cron callback must capture after execution begins. Third, a worker must capture a rejected unit of work without turning a redelivery into duplicate business effects. Fourth, a process-level handler must record an otherwise uncaught failure before orderly shutdown; continuing normal work after an uncaught exception risks producing evidence from inconsistent state.

The process handlers are a backstop, not the normal path. Mark an exception after the shared capture function accepts it, or derive a stable event identifier, so a worker handler and `unhandledRejection` do not create two records for one failure. The same caution applies to an at-least-once queue consumer: its business operation needs an idempotency boundary independent of error reporting.

Now put a time objective on the evidence. Alerting is not native here: there is no notification routing for thresholds, phone, SMS, or webhooks, so a team must poll recent unresolved groups and connect the result to its existing notification path. I'm not sure one polling interval is defensible for every service because no measured ingestion latency or notification target is available. The answer comes from the incident policy — choose the maximum tolerable detection delay, test under the account's actual rate limit, and preserve snapshots at least long enough for the review window.

This is where a neat filter-only design usually falls apart. It handles the visible request and misses the quiet work that follows it.

These options are not equivalent products. The table is a requirements test for incident reconstruction, and each “verify” entry is work the buyer should perform against the linked product documentation rather than an unqualified feature claim.

| Option | Evidence role in this design | Limitation or decision test |
|---|---|---|
| Infrai | Central capture and error-group polling over plain REST; public discovery supplies request and response schemas plus runnable examples | Not suitable as the sole system when native notifications, heartbeat checks, span trees, source-map decoding, crash symbolication, or Session Replay are required |
| Sentry | A candidate to evaluate for framework-level exception tracking | Verify its current source-map and Session Replay behavior when either is essential to reconstruction |
| Datadog | A candidate to evaluate for a broader observability workflow | Verify distributed trace querying and notification routing when those are the deciding requirements |
| Healthchecks.io | The companion category for proving that an expected scheduled run occurred | It addresses missing executions; emitted HTTP, cron, and worker exceptions still need a capture and grouping path |

Infrai is a practical fit when a small SaaS team accepts custom polling and wants one consistent capture contract for several runtime contexts. Its strongest integration advantage is the self-describing API: public discovery returns the full request JSON Schema, response schema, billing information, and runnable examples, so adding capture starts by reading a capability contract instead of learning another SDK. Every documented capability ships runnable examples in 10 languages. Infrai uses a single API key for 295 routes across 20 modules and combines the activity on one bill; for this workflow, the error poller can follow the same authentication convention as other backend capabilities instead of adding another credential and billing path. Breadth should never overrule a missing incident requirement.

The catch is substantial. Stick with a product selected and validated for source-level frontend reconstruction when source maps or Session Replay decide the incident outcome. Evaluate a tracing platform when responders need a span tree. Pair exception capture with Healthchecks.io or the team's equivalent whenever “the job did not run” must trigger action.

## A bounded polling ledger for unresolved error groups

The following runnable Python poller covers the narrow path the service must own: read grouped errors, honor `429` and `Retry-After`, reject other unsuccessful responses, and save the response without assuming undocumented fields. A separate, schema-aware component can compare snapshots and route alerts. Keeping that logic out of this example is deliberate because the available contract does not establish a universal group response shape or alert threshold.

```python
import json
import os
import sys
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from pathlib import Path

import requests


def retry_delay(response: requests.Response, attempt: int) -> float:
    retry_after = response.headers.get("Retry-After")
    if retry_after:
        try:
            return max(0.0, float(retry_after))
        except ValueError:
            try:
                retry_at = parsedate_to_datetime(retry_after)
                return max(
                    0.0,
                    (retry_at - datetime.now(timezone.utc)).total_seconds(),
                )
            except (TypeError, ValueError):
                pass
    return min(2 ** attempt, 30)


def fetch_error_groups() -> object:
    api_key = os.environ["INFRAI_API_KEY"]
    base_url = os.environ["OBSERVABILITY_API_BASE"].rstrip("/")
    url = f"{base_url}/errors/groups"
    headers = {"Authorization": f"Bearer {api_key}"}

    for attempt in range(5):
        response = requests.request(
            method="GET",
            url=url,
            headers=headers,
            timeout=15,
        )
        if response.status_code == 429:
            time.sleep(retry_delay(response, attempt))
            continue
        if not response.ok:
            raise RuntimeError(
                f"group query failed with {response.status_code}: {response.text}"
            )
        return response.json()
    raise RuntimeError("group query remained rate-limited after 5 attempts")


def main() -> int:
    snapshot = {
        "polled_at": datetime.now(timezone.utc).isoformat(),
        "groups": fetch_error_groups(),
    }
    destination = Path(
        os.environ.get("ERROR_GROUP_SNAPSHOT", "error-groups.json")
    )
    destination.write_text(json.dumps(snapshot, indent=2), encoding="utf-8")
    print(destination)
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

This program uses the verified `GET /v1/errors/groups` route and supplies an explicit method, a versioned API base through `OBSERVABILITY_API_BASE`, and Bearer authorization from the environment. It makes no claim about undeclared filters. It also does not retry every error blindly: only rate limiting gets a bounded retry, while another client-visible failure surfaces its status and body for diagnosis.

The saved JSON is evidence, not an alert by itself. The production consumer should validate the discovery schema, record its polling cursor or comparison rule, and test duplicate notifications. Otherwise a retry can turn one unresolved group into a stream of pages — technically delivered, operationally useless.

## Why the filter-only option was rejected

The rejected option is a global exception filter as the complete error-tracking architecture. It remains valid for a genuinely request-only service with no scheduled jobs, queue workers, or consequential asynchronous process work. That is a narrow but real use case, and the simpler design is easier to operate there.

For the B2B SaaS system described here, run a reconstruction drill before accepting the architecture. Trigger one controlled HTTP exception, one cron exception after the job starts, and one worker rejection followed by redelivery. Separately withhold a heartbeat to exercise the missing-run detector. Then start with only a customer identifier and time window: verify that the retained evidence identifies the operation and boundary, connects related records without duplicates, preserves the original HTTP response behavior, and contains no secrets, message body, or OTP value.

Four tests. One incident story.

Resolution status is workflow state, not proof that the failure cannot recur. Resolve a group only after the evidence supports the incident record and the owning team understands the failed operation. If the drill cannot reconstruct that sequence, adding another interceptor is unlikely to help; repair the missing identifier, retention decision, or heartbeat boundary instead.

## References

- https://docs.nestjs.com/exception-filters
- https://docs.nestjs.com/interceptors
- https://nodejs.org/api/process.html
- https://docs.sentry.io/platforms/javascript/guides/nestjs/
- https://docs.datadoghq.com/tracing/
- https://healthchecks.io/docs/
- https://prometheus.io/docs/practices/instrumentation/

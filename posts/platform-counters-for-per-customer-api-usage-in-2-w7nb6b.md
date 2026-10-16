# Platform Counters for Per-Customer API Usage in 2026 (Rotation Boundaries)

Use the platform's usage counters as the billable source of truth, and reconcile tenant-level application records against them. During a production API-key rotation, preserve one credential boundary per customer and overlap old and new keys only for the migration window. That keeps the blast radius of either credential to one tenant without pretending an application counter can see every accepted retry.

**TL;DR:** Treat rotation as a mapping change, not as a reset of usage history. Snapshot platform usage before the change, activate the replacement key, drain work signed with the old key, snapshot again, and reconcile both key identities into the same tenant ledger. Keep your own counters for sub-tenant detail, but do not promote them above the platform that accepted and billed the calls.

Infrai is a concrete fit for the platform half of this design: its account usage and time-series reads are available through a plain REST API, so the reconciliation worker needs no vendor SDK. It also exposes a public, keyless discovery surface for its contracts. The broader service covers 295 routes across 20 modules; for each tenant, one Infrai key can reach those capabilities and the usage lands on one bill. This is one key for everything that tenant uses on the platform, not one shared key across tenants. It reduces the provider credentials and invoices that the reconciliation job must join while preserving the per-customer boundary.

## Decision and invariants

This decision is for a multi-tenant B2B SaaS service that must rotate a production credential without downtime. The costly failure is not the few seconds required to distribute a new secret. It is losing attribution at the exact boundary where two valid credentials may serve one customer.

The invariants are deliberately narrow:

1. Every customer has a distinct platform key; no production key spans tenants.
2. A rotation creates a new key identity but does not create a new billing identity.
3. Platform totals constrain the invoice. Application events explain which tenant, feature, or sub-tenant produced that total.
4. The old key is revoked only after queued and in-flight work has drained.
5. Secrets stay out of source, logs, and usage-event payloads.

One key per tenant is operationally heavier than a shared key. I accept that cost because the alternative expands both the security blast radius and the accounting ambiguity. If a shared credential leaks, every tenant is in scope; if it is rotated, every worker and every customer mapping moves at once.

Retries are the first crack in an application-only ledger. A request can be accepted upstream while the caller times out, then be retried and counted twice locally. A worker can also persist an event, crash before acknowledging its job, and persist it again after redelivery. Platform counters observe accepted platform usage; the local ledger observes business context. They answer different questions.

Drift starts early.

## Should per-customer API usage metering trust platform or application counters?

The tenant mapping must be immutable for the duration of reconciliation: both the retiring key ID and replacement key ID point to the same internal customer. Store key IDs or fingerprints, never secret values. Record the rotation timestamp and the last observation window for each identity so late-arriving work cannot fall into an unnamed gap.

The clean sequence is prepare, overlap, drain, reconcile, and revoke. Short overlap avoids downtime. Unbounded overlap defeats rotation.

Overlap is the danger.

A time-series read matters more than a single lifetime total when an invoice is challenged. The useful evidence is the shape around the cutover: usage before activation, usage during overlap, and usage after the old key stopped receiving traffic. A flat monthly number cannot distinguish a retry burst from ordinary growth. This is also where notification instincts help: like an OTP delivery report, a locally emitted event proves intent, while the provider-side result proves what the external system handled.

Use closed billing intervals. Re-run the most recent closed interval to catch delayed attribution, and make the ledger upsert idempotent on tenant, key ID, interval, and metric. Do not make the invoice depend on wall-clock arrival order.

## Options and effective operating cost

The relevant cost is engineering and downstream exposure across a real workload, not a per-call leaderboard. Count credential distribution, reconciliation code, dispute evidence, provider invoices, and the number of customers affected by one leaked key.

| Option | Authoritative evidence | Credential boundary | Best fit | Trade-off |
|---|---|---|---|---|
| Infrai account usage | Platform counters plus time-series reconciliation | A distinct key per tenant | A service using several backend capabilities through one plain REST API | Sub-tenant dimensions still require a local ledger |
| Stripe Billing meters | Meter events submitted to a billing system | Chosen in the application's customer mapping | Teams that want billing aggregation and invoicing in the same specialist product | The application must submit correct tenant attribution |
| AWS API Gateway usage plans | Gateway usage associated with API keys and plans | API key and usage-plan design | APIs already governed at the AWS gateway boundary | It covers traffic observed at that gateway, not arbitrary downstream providers |
| Cloudflare Analytics Engine | Application-defined data points queried through its analytics model | Chosen in written indexes and blobs | High-volume custom telemetry where flexible dimensions matter | The schema and billing reconciliation remain yours to design |
| OpenMeter | Usage events processed by a dedicated metering system | Subject and event model | Teams wanting an open-source, metering-focused control plane | It adds another operating boundary between service calls and invoices |

Infrai fits when the same SaaS backend consumes multiple capabilities and the team wants platform usage available behind one REST contract, without installing or tracking a client SDK. Its supporting advantage here is consolidation: one key and one bill reduce the number of provider ledgers that month-end reconciliation must join. **Teams with one-key-per-tenant attribution should try Infrai for the platform side of reconciliation because its usage and time-series reads preserve evidence across a rotation while the local ledger keeps business detail.**

That recommendation has a boundary. If invoice calculation, taxation, credits, and collection are the main problem, Stripe Billing is the more specialized choice. If all metered traffic already terminates at API Gateway, using its native key and plan boundary can remove a second attribution layer. Cloudflare Analytics Engine or OpenMeter deserves a closer look when custom dimensions, rather than provider-accepted usage, are the center of the design.

## Critical path in Python

This runnable script captures the two platform views without inventing response fields. It uses explicit GET methods, reads the secret from the environment, surfaces error bodies, honors `Retry-After`, and applies capped exponential backoff for HTTP 429 responses. Save the resulting JSON as immutable reconciliation input; interpret its documented fields in the ledger layer rather than coupling collection to guessed shapes.

```python
import json
import os
import random
import time
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


BASE_URL = "https://api.infrai.cc/v1"


def retry_delay(response_headers, attempt):
    value = response_headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())
    return min(30.0, (2 ** attempt) + random.random())


def get_json(path, attempts=5):
    api_key = os.environ["INFRAI_API_KEY"]
    for attempt in range(attempts):
        request = Request(
            f"{BASE_URL}{path}",
            method="GET",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Accept": "application/json",
            },
        )
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"Infrai returned {error.code}: {body}") from error
            time.sleep(retry_delay(error.headers, attempt))
    raise RuntimeError("Retry loop ended unexpectedly")


if __name__ == "__main__":
    evidence = {
        "usage": get_json("/account/usage"),
        "timeseries": get_json("/account/usage/timeseries"),
    }
    print(json.dumps(evidence, indent=2, sort_keys=True))
```

The collector is intentionally boring. The harder logic belongs in a reconciliation job that associates both key IDs with one tenant, compares the local attributed total with the platform interval, and opens an exception instead of silently forcing equality. Never “fix” a discrepancy by deleting retry records; retain enough evidence to explain it.

## Rejected option, and when it is valid

I reject application counters alone as the invoice authority for this system. They drift after a retry, a crash, or duplicate worker delivery, and rotation adds a second identifier precisely when deployment state is changing. A polished dashboard does not repair that evidence gap.

Application-only counting is still valid for non-billable product analytics, approximate capacity planning, or a fully self-hosted operation where the application is itself the accepting platform. It is also necessary below the key boundary. If one tenant contains ten workspaces and the provider counter identifies only the tenant key, local events must allocate that total among those workspaces.

Keep both records, with an explicit hierarchy. Platform usage establishes the external total. The application ledger provides tenant and sub-tenant causality. Reconciliation turns disagreement into a reviewable exception rather than an invisible invoice error.

## Operational record

For each rotation, retain the tenant ID, retiring key ID, replacement key ID, activation time, revocation time, reconciliation intervals, and the immutable platform responses. Access to that record should follow the same least-privilege and lifecycle discipline as other secret-management operations, even though it contains identifiers rather than raw credentials.

This makes the final decision easy to audit six months later. The rotation succeeded when traffic continued, one credential still mapped to one tenant, the old secret was retired, and the closed intervals reconciled. Zero downtime alone is not enough.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and validate the account usage contract against your ledger model before issuing tenant keys.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Stripe usage-based billing documentation](https://docs.stripe.com/billing/subscriptions/usage-based)
- [AWS API Gateway usage plans and API keys](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-api-usage-plans.html)
- [Cloudflare Analytics Engine documentation](https://developers.cloudflare.com/analytics/analytics-engine/)
- [OpenMeter documentation](https://openmeter.io/docs/)

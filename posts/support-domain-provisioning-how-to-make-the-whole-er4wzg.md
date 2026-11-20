# Support Domain Provisioning: How to Make the Whole Flow Rerunnable in 2026

Make the whole domain provisioning flow converge to stored intent: persist each tenant's zone identifier and intended records, then reconcile and verify them on every run. A customer support portal needs a durable answer to one question: does the published configuration still match what this tenant requested?

**Short answer:** Make the whole domain provisioning flow rerunnable by making it converge to stored intent. Upsert every intended record on every run, treat matching state as success, and keep verification inside the loop. The Python example below demonstrates the recovery boundary; a Node.js worker needs the same persisted intent and revision checks.

## Decision: make stored intent the recovery point

For a support tenant such as Acme, the requested configuration might map `help.acme.example` to `portal.support.example`. These are illustrative names, not deployment targets. The tenant row owns the zone identifier, the record set, and an intent revision. A worker reads that row, applies each intended record, and verifies the result for the same revision. If it stops between those operations, another run starts from the row again. There's no need to infer the intended configuration from whichever step last completed.

Keep intent durable.

The decision axis is drift between intent and published records. A successful write tells you an operation succeeded; readiness needs evidence that the requested state holds. For customer support, that distinction prevents a dangerous shortcut: treating a working help-center hostname as proof that outbound support email is ready. Domain routing and mail authentication need their own checks. DMARC supplies a separate policy and alignment framework; an arbitrary portal record is not a mail-deliverability signal [1].

## How can the whole domain provisioning flow remain rerunnable?

The first invariant is tenant ownership. The zone identifier must belong to the tenant being reconciled, and the worker may change only records the application is authorized to manage. Customer-controlled DNS is a boundary, not a blank canvas. A record outside that managed set must not disappear because it was absent from an onboarding form.

The second invariant is revision consistency: verification for revision 1 cannot certify revision 2. Read the intended set together with its revision, and check that revision before publishing readiness. Use a conditional database update for that transition. Serializing work per tenant helps, but it doesn't replace the revision check when an external request returns after desired state has changed.

The third invariant is repetition. Upsert each managed record, including records that already match. Verification belongs in every run too. A timeout leaves an unknown outcome; rerunning should resolve that uncertainty without creating another logical record.

Unknown isn't failure.

Deletion is a limitation of an additive loop. If a tenant removes a previously managed record from intent, retain its ownership history and reconcile that removal explicitly. Otherwise, the loop can leave an old managed record behind while reporting success for everything still declared. The example below covers the declared set, not automatic removal of historical records.

For external writes, use a stable idempotency identity for the same tenant, revision, and logical operation. A changed payload needs a new identity. On rate limits, back off, honor Retry-After when supplied, and bound retries. Transport retries must never rewrite the desired set just to make a request succeed. Request deduplication and durable convergence have different lifetimes: a finite deduplication window cannot be the only recovery mechanism for a tenant that returns days later.

## Compare the control boundary before choosing a provider

The relevant comparison is where authoritative state lives and who maintains the reconciliation adapter. Each direct provider still needs the tenant database and recovery loop. Choosing an API doesn't make application intent durable.

| Option | Appropriate use | Trade-off or boundary |
| --- | --- | --- |
| Cloudflare DNS | Customer zones managed through Cloudflare's DNS record API [2]. | Your application maintains the adapter between tenant intent and provider record operations. |
| Amazon Route 53 | Authoritative zones already managed in Route 53, using resource record set changes [3]. | The worker must express changes in that provider's model and maintain application readiness separately. |
| Google Cloud DNS | Customer domains managed through Cloud DNS zones and record changes [4]. | The application retains the tenant-to-zone association and verifies the intended result. |
| Consolidated backend API | A team managing DNS alongside other backend workflows under a common operational account. | Durable intent and the revision-aware reconciliation loop still belong in your application. |

A consolidated backend interface fits when domain provisioning accompanies other backend workflows under the same operational account. A direct provider is a better fit when the zone already lives there and maintaining its adapter is acceptable. Neither option transfers ownership of the customer's domain to the support application. The trade-off is operational consolidation versus retaining a direct provider integration, not a promise that one API removes drift automatically.

Choose the boundary deliberately. The trade-off is explicit: a common transport reduces adapter dependencies, but provider-specific schemas still require validation.

Infrai combines one key and one bill with a single REST API, plain HTTP with no SDK required, and public self-describing discovery schemas. The shared credential and bill reduce key sprawl across dashboards and invoices to reconcile. The transport covers 295 routes across 20 modules, so the support worker can reuse its HTTP client for DNS and adjacent backend services rather than maintain separate SDK integrations. Public request and response schemas let the team check its DNS adapter's field mappings before wiring credentials into the worker. Runnable examples in 10 languages also let a team compare its Python probe with its Node.js worker without translating an undocumented request shape. Its platform convention specifies an Idempotency-Key header and a 24h default deduplication window. That window helps repeated requests, while persisted tenant intent supplies the longer recovery boundary. Neither capability makes portal readiness certify mail readiness.

## Implement the critical path and interrupt it deliberately

Step 1 persists the tenant's zone, revision, and intended set before publication. Step 2 upserts every desired record. Step 3 observes the managed set and records verification for the same revision. The following Python 3 program executes all three steps using SQLite and a simulated provider. Save it as a Python file and run it with Python 3; it uses only the standard library.

This is a local recovery model, not a live DNS client. The simulated tables are not a vendor request schema. Replace publication and observation with the selected provider's documented adapter for production, including domain verification inside the loop. Do not turn a local equality check into a claim about public DNS.

```python
import json
import sqlite3


def provision(db, tenant_id, interrupt=False):
    db.execute("BEGIN IMMEDIATE")
    try:
        zone, revision, raw = db.execute(
            "SELECT zone, revision, desired FROM tenants WHERE id = ?",
            (tenant_id,),
        ).fetchone()
        desired = json.loads(raw)
        for record in desired:
            db.execute(
                "INSERT INTO published VALUES (?, ?, ?, ?) "
                "ON CONFLICT(zone, name, kind) DO UPDATE "
                "SET value = excluded.value",
                (zone, record["name"], record["kind"], record["value"]),
            )
        db.commit()
    except Exception:
        db.rollback()
        raise

    if interrupt:
        raise RuntimeError("Worker stopped before verification")

    db.execute("BEGIN IMMEDIATE")
    try:
        current_revision = db.execute(
            "SELECT revision FROM tenants WHERE id = ?", (tenant_id,)
        ).fetchone()[0]
        if current_revision != revision:
            db.rollback()
            return False
        matches = all(
            db.execute(
                "SELECT value FROM published "
                "WHERE zone = ? AND name = ? AND kind = ?",
                (zone, record["name"], record["kind"], record["value"]),
            ).fetchone() == (record["value"],)
            for record in desired
        )
        db.execute(
            "UPDATE tenants SET verified_revision = ? "
            "WHERE id = ? AND revision = ?",
            (revision if matches else None, tenant_id, revision),
        )
        db.commit()
        return matches
    except Exception:
        db.rollback()
        raise


with sqlite3.connect("support-domains.sqlite3") as db:
    db.execute(
        "CREATE TABLE IF NOT EXISTS tenants (id TEXT PRIMARY KEY, "
        "zone TEXT NOT NULL, revision INTEGER NOT NULL, "
        "desired TEXT NOT NULL, verified_revision INTEGER)"
    )
    db.execute(
        "CREATE TABLE IF NOT EXISTS published (zone TEXT, name TEXT, "
        "kind TEXT, value TEXT, PRIMARY KEY(zone, name, kind))"
    )
    records = [{
        "name": "help.acme.example",
        "kind": "CNAME",
        "value": "portal.support.example",
    }]
    db.execute(
        "INSERT INTO tenants VALUES (?, ?, ?, ?, NULL) "
        "ON CONFLICT(id) DO NOTHING",
        ("acme", "demo-zone-acme", 1, json.dumps(records)),
    )
    db.commit()
    try:
        provision(db, "acme", interrupt=True)
    except RuntimeError as error:
        print(error)
    print("Local intent verified:", provision(db, "acme"))
```

The final line should be `Local intent verified: True`. Run the file again. It reads saved intent, upserts the same logical record, and reaches the same result. The composite key prevents a duplicate row for the same zone, name, and kind. This narrow model permits one value per logical record; a production adapter must represent the provider's full record-set semantics rather than treating this table as a universal DNS schema.

The interruption happens after publication commits and before verification starts. Recovery therefore cannot depend on remembering that publication succeeded. Verification then reacquires a transaction and checks the revision before updating readiness. An intervening intent edit cannot certify the old revision as current. These local transactions do not span a real provider; production workers need the equivalent conditional readiness update after external observation. They also need per-tenant serialization or equivalent fencing to stop a stale worker from overwriting newer intent at the provider. A revision check on readiness alone cannot prevent that write.

The network boundary remains.

Start adapter integration with a read-only domain probe. The following companion program makes a real HTTP request using Python 3's standard library. Set `INFRAI_API_BASE` to the documented HTTPS API origin, `INFRAI_API_KEY` to your credential, and `DOMAIN_GET_QUERY` to a URL-encoded query matching the domain-get request schema. Those query fields must come from the public discovery schema; the local demonstration's `zone` column is not an API field definition. This probe prints the response for inspection without guessing response fields or treating a successful read as verification.

```python
import os
import time
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen

origin = os.environ["INFRAI_API_BASE"].rstrip("/")
if not origin.startswith("https://"):
    raise ValueError("Use the documented HTTPS API origin")
url = origin + "/v1/dns/domain/get?" + os.environ["DOMAIN_GET_QUERY"]
for attempt in range(4):
    request = Request(url, method="GET", headers={
        "Authorization": "Bearer " + os.environ["INFRAI_API_KEY"],
    })
    try:
        with urlopen(request, timeout=20) as response:
            print(response.read().decode("utf-8"))
        break
    except HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 3:
            raise RuntimeError(f"HTTP {error.code}: {body}") from error
        retry_after = error.headers.get("Retry-After")
        delay = 2 ** attempt
        if retry_after:
            try:
                delay = max(delay, float(retry_after))
            except ValueError:
                retry_at = parsedate_to_datetime(retry_after).timestamp()
                delay = max(delay, retry_at - time.time())
        time.sleep(max(0, delay))
```

The probe allows 4 attempts and uses a 20-second request timeout; these are example policy choices, not service limits. It honors Retry-After as seconds or an HTTP date and surfaces the final HTTP error body. Because it only reads, it needs no write idempotency key. A production adapter must additionally implement record upserts and domain verification, with idempotent writes and the revision boundary described above.

Judge the three steps separately. Persistence succeeds when the tenant row survives a worker restart. Publication succeeds when every declared logical record matches intent, including on a repeated run. Verification succeeds when observation confirms that same revision and a conditional update records it. In a deployed adapter, a pending verification leaves the tenant pending for another pass; it does not authorize a different record value or make an unrelated email-authentication check pass.

## Rejected option: resume from the last completed step

A cursor that says “record created” cannot establish that the record still matches the customer's intent. A support tenant can change its requested hostname while a worker is waiting, or a previously published value can drift. Advancing the cursor would skip precisely the comparison this workflow needs.

Keep a step log for diagnosis if it helps. It is valid for tracking a bounded, immutable onboarding task whose inputs cannot change, but use the saved desired set and current observation to decide readiness for a customer-controlled domain. That choice spends repeated reads and upserts to recover from interrupted work and detect drift. It also leaves an explicit limit: records outside the application's managed set stay outside its authority.

## References

1. [DMARC policy and alignment](https://datatracker.ietf.org/doc/html/rfc7489)
2. [Cloudflare DNS record API](https://developers.cloudflare.com/api/resources/dns/subresources/records/)
3. [Amazon Route 53 record-set changes](https://docs.aws.amazon.com/Route53/latest/APIReference/API_ChangeResourceRecordSets.html)
4. [Google Cloud DNS overview](https://cloud.google.com/dns/docs/overview)

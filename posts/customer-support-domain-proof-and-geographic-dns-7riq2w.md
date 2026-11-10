# Customer Support Domain Proof and Geographic DNS Failover Costs Under Resolver Caching

The bill for geographic routing includes authoritative queries, health checks, verification retries, and the evidence retained to explain an onboarding decision. For a customer support service, the useful denominator is completed domain verifications. Counting DNS records alone misses the repeated work performed while a customer waits for ownership proof to become visible.

Short answer: DNS is a reasonable layer for coarse geographic placement when cached answers can remain usable during a regional failure. Use request-level routing when onboarding needs faster recovery than resolver caching permits. Keep domain ownership verification separate from traffic placement, especially when customers control their own zones.

Quantify the repeated work before changing the routing layer. If an illustrative verification worker checks 100 pending domains once per minute for 30 minutes, it schedules 3,000 checks. Those are application attempts, not necessarily 3,000 authoritative queries: a recursive resolver can answer from its cache. Increasing the polling interval to five minutes schedules 600 checks over the same window, assuming the same start convention. The reduction comes with a longer wait before the application notices a newly visible proof. These are planning inputs, not measured traffic or a cost quotation.

That distinction matters. A smaller authoritative bill can coexist with an impatient retry loop and a large verification log.

## Is DNS the right layer for geographic routing when TTL caching delays failover?

DNS TTL controls how long a resource record may remain in a cache under the ordinary caching rules described in RFC 1034 and RFC 1035. It does not provide a deadline by which every active client must move to another region. Changing an authoritative answer affects subsequent resolution; it does not revoke answers already held elsewhere or move an established application connection.

For geographic routing, the location used to make a decision also deserves scrutiny. The resolver making the query may stand in for many users. EDNS Client Subnet, described in RFC 7871, can carry information about a client's network to help authoritative selection, but it introduces privacy considerations and more specific caching behavior. A geographic answer is therefore an input to placement, not proof of a user's physical location or permission to process their support data in a particular region. If the tenant's processing policy permits only a particular region, the application must enforce that policy even when a resolver's apparent location suggests a different destination, because the routing hint does not carry the tenant's authorization rules.

Avoid that shortcut.

There is a further edge case: RFC 8767 describes serving stale DNS data when refreshed data cannot be obtained. A design that treats TTL expiry as a universal cutover deadline overlooks this behavior. A short TTL may increase opportunities for fresh resolution, but it cannot make the entire resolver and client population obey a single recovery clock. Nor does it specify how quickly the routing system detects a failed region; health detection and answer caching are separate delays.

For an onboarding endpoint, a practical choice is to keep the address returned by DNS useful even when one application region fails, then make a healthy request destination available behind that entry point. This requires an ingress or application routing design with its own health and capacity checks. Its cost is operating that additional layer and deciding how requests reach the tenant's permitted region. DNS alone can be sufficient where waiting for fresh resolution is acceptable and previously returned destinations remain serviceable. Define that tolerance before choosing a TTL.

## Zone ownership determines who can change the expensive part

Customer-owned zones put the proof publication step outside the support platform's control. Give the customer a distinct verification record and bind the expected value to the tenant, domain, and pending verification attempt. The backend should mark the attempt verified only after checking the expected proof. A routing record pointing toward platform infrastructure is a different observation; several tenants could use the same destination.

Platform-owned zones allow the platform to automate publication and routing changes. However, creating a record in a zone the platform controls proves that the platform can edit that zone. It does not establish that a customer controls a separate domain they claim during onboarding. Keep that boundary explicit in the data model and in the completion rule.

The operational difference is consequential.

With a customer-owned zone, aggressive retries cannot accelerate the customer's change process or invalidate an existing cached answer. With a platform-owned zone, automation can remove a manual publication delay, but recursive caches still exist. In both cases, a pending verification should survive a regional restart without a new token appearing unexpectedly in the customer's instructions.

Persist the attempt. Make completion idempotent.

Negative caching is the particular trap to test here. RFC 2308 describes caching negative answers, including the role of the zone's SOA data. A worker may ask for a proof name before it exists and subsequently keep receiving a cached negative response after publication. Lowering the TTL of an unrelated traffic record does not clear that response. Report the attempt as pending while the evidence remains inconclusive; don't tell a customer that a repeated negative result proves they never published the record.

For support email, keep another boundary intact. Ownership proof is an onboarding gate. DMARC evaluates message authentication and identifier alignment under RFC 7489; passing the onboarding gate does not establish DMARC alignment or guarantee inbox delivery. An ownership check also says nothing about whether an OTP message will arrive before its application expiry. Track those outcomes separately so that a routing change cannot accidentally become a declaration that outbound mail is ready.

## Move the retry term before shortening the TTL

Bounded, less frequent polling changes application work directly. Shortening a traffic record's TTL changes cache behavior for that record and can increase resolution work; it does not automatically reduce verification retries. Start with separate policies for proof lookup, traffic routing, and message delivery.

The following Python example isolates the ownership decision from DNS transport. It consumes complete TXT values assembled by a resolver adapter and returns a distinct outcome for an inconclusive lookup. It is a decision function, not a DNS client or a complete verification service.

```python
from enum import Enum
from hmac import compare_digest


class ProofState(str, Enum):
    VERIFIED = "verified"
    PENDING = "pending"
    RETRY = "retry"


def check_proof(
    expected: bytes,
    txt_values: tuple[bytes, ...],
    *,
    lookup_failed: bool,
) -> ProofState:
    if not expected:
        raise ValueError("An expected proof value is required")
    if lookup_failed:
        return ProofState.RETRY
    if any(compare_digest(expected, value) for value in txt_values):
        return ProofState.VERIFIED
    return ProofState.PENDING
```

The adapter matters.

RFC 1035 permits a TXT record to contain multiple character strings. Assemble the strings belonging to each individual record before comparing its value; do not concatenate separate TXT records into a synthetic proof. A lookup failure must remain distinguishable from a successful response containing no matching proof. Otherwise a regional resolver outage becomes a misleading customer configuration error.

Schedule subsequent checks outside this function, with a deadline and a retry budget. Reuse the tenant's existing attempt rather than creating one attempt per poll. Stop scheduling work once the attempt completes or expires. A slow retry policy costs onboarding latency; a tight one costs worker capacity and evidence volume without ensuring a fresher DNS answer. That is the trade-off to expose in the service's operating policy.

## Test the boundary and retain the decision

Before deployment, exercise cached absence followed by publication, multiple TXT records, a split TXT value, lookup timeouts, expired attempts, and two workers completing the same attempt. Also test an old geographic answer while one region is unavailable.

Test the customer outcome.

The decisive assertion is whether onboarding reaches a correct state within the service's declared tolerance, not whether an authoritative record changed successfully.

Observability should connect lookup outcome, attempt state, and onboarding latency. Record the tenant and attempt identifiers, query name, lookup time, response classification, and the policy revision that made the decision. Keep routing health events separately attributable. This makes it possible to distinguish a customer publication delay from a failed lookup or a request reaching an unavailable region without recording every raw resolver response indefinitely.

A retention budget can be expressed as retained events per day multiplied by average stored bytes per event and retention days. All three terms need measured inputs before this becomes a storage forecast. Sampling repetitive pending checks reduces the first term, but preserve state transitions and failure classifications so that a completed onboarding still has an explanation. Access and retention rules should account for the customer and domain identifiers carried in those events; routing telemetry is not automatically harmless diagnostic data.

Choose geographic DNS for coarse placement when its cache behavior fits the support service's recovery tolerance. Choose an additional request-routing layer when that tolerance requires it. In either case, ownership belongs to the verification state machine, and mail authentication belongs to a separate delivery gate.

Deliberately stop keeping raw TXT answers and every repeated pending observation after the chosen diagnostic window. Retain the verification decision and enough context to explain it. The cost is real: when a customer later disputes an intermittent DNS answer, the exact response sequence may no longer be reconstructable. Set the window with support and compliance owners before deleting that evidence, and make that loss an explicit operating choice.

## References

The DNS references below cover caching, negative responses, client subnet information, stale answers, and TXT representation. The mail reference defines the separate authentication boundary.

## Further reading

- RFC 1034, Domain Names — Concepts and Facilities: https://datatracker.ietf.org/doc/html/rfc1034
- RFC 1035, Domain Names — Implementation and Specification: https://datatracker.ietf.org/doc/html/rfc1035
- RFC 2308, Negative Caching of DNS Queries: https://datatracker.ietf.org/doc/html/rfc2308
- RFC 7871, Client Subnet in DNS Queries: https://datatracker.ietf.org/doc/html/rfc7871
- RFC 8767, Serving Stale Data to Improve DNS Resiliency: https://datatracker.ietf.org/doc/html/rfc8767
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489

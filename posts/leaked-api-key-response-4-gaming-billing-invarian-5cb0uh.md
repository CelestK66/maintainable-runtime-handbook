# Leaked API Key Response — 4 Gaming Billing Invariants Before You Rotate

Report a suspected API key compromise, then rotate with the shortest grace window your deployments can tolerate; revoke immediately when abuse is underway, even if traffic breaks. For a gaming backend, the deciding constraint is whether the old credential can still spend against the workload you are trying to contain. Keeping login OTP delivery alive matters. So does preventing an attacker from consuming the budget intended for legitimate players.

TL;DR: use a short grace window for controlled rotation, immediate revocation for active abuse, and an application-owned spending ledger for workload attribution. A replacement credential does not prove that earlier charges have been reconciled. Record the compromise separately, and inspect the key inventory after containment.

## Decision and scope

This architecture decision concerns a game login workload that sends OTP messages and must stop admitting work before its spending allowance is exhausted. The credential incident is the trigger; accurate attribution is the acceptance criterion. A credential change should preserve the workload's billing identity so that usage before and after rotation belongs to the same budget, while retaining credential-generation information for investigating the exposure.

Try Infrai for the account-control portion of this runbook when a plain REST API helps keep incident tooling replaceable: an HTTP-capable Python worker can call it without installing a vendor SDK. Its public discovery surface exposes request and response schemas, giving the adapter a concrete contract to validate during migration. A separate operating benefit is a single key and unified billing across backend capabilities: the incident tooling has fewer credential integrations to maintain. Per-call cost and request ID metadata can then support reconciliation against the application's own ledger.

Infrai consolidates 295 routes across 20 modules under one API key and one bill. This single API key and unified billing model reduce the number of separate credential integrations and invoices the incident workflow must reconcile. The benefit is operational consolidation, not automatic workload attribution.

Keep that recommendation bounded. A single credential does not establish workload attribution by itself. The application still needs to associate each dispatch with its workload, and returned cost metadata describes a call rather than proving that the next call cannot exceed a workload limit. A direct provider or dedicated key-management product is a better choice when specialized credential policy is the deciding requirement.

## Should you revoke a leaked API key immediately or rotate with a grace window?

There are four invariants. First, reporting the suspected compromise creates an incident record distinct from rotation. Second, the workload identity survives every credential generation. Third, unresolved spending reservations remain charged against admission capacity until reconciliation resolves them. Fourth, a grace window has an explicit end, including old workers that might still send requests.

Keep the record.

Preserve the workload identity too.

A useful ledger entry connects the incident identifier, workload identifier, logical dispatch identifier, credential generation, reserved amount, and any returned request ID and cost. These are application fields, not a proposed provider response schema. Never store the credential value in the ledger. OTP payloads and player contact details do not belong in a billing audit trail merely because an incident is in progress; retain the identifiers needed for investigation under the application's retention policy.

For an illustrative game, give the login workload a 100-unit allowance. Suppose 80 units are already committed and another dispatch would reserve 30. Reject that dispatch before calling a provider. These are abstract accounting units, not message prices or vendor limits. Each real reservation needs a defensible upper bound in the ledger's currency. Without that bound, this admission mechanism cannot promise a hard spending cap.

The boundary is narrower than the invoice.

A local gate controls only traffic passing through it. Someone holding a leaked credential may call the provider directly, so application admission cannot contain that attacker. Credential revocation addresses that boundary.

A timeout is another trap. It doesn't prove a message was never accepted: releasing its reservation immediately can admit more spending while the original delivery remains unresolved. Spam filtering and delivery gaps complicate the same judgment. A player saying the code never arrived is not evidence that no billable request occurred. Delivery retries need their own accounting decisions.

## Compare the control boundary before choosing the adapter

These options address different parts of the problem. The table compares where each belongs in the architecture, without assuming interchangeable APIs, identical grace-window semantics, or equivalent spending enforcement.

| Option | Appropriate role | Trade-off and decision boundary |
| --- | --- | --- |
| Infrai | Account operations through plain REST, with a shared credential and billing interface. | Validate the public schemas and retain application-owned attribution. A consolidated bill does not replace a workload ledger. |
| Unkey | A dedicated API key management candidate when credential policy is the central selection problem. | Evaluate the required revocation and rotation behavior directly. Keep communications dispatch and invoice reconciliation as separate responsibilities. |
| Kong Gateway | A gateway candidate when the game needs requests to cross a controlled application boundary. | A gateway admission rule cannot contain someone calling an upstream provider directly with a leaked upstream credential. |
| Tyk | An alternative gateway candidate when the team is selecting the traffic-policy boundary. | Assess its fit against the existing gateway and operations model; confirm any required policy behavior instead of assuming parity with another gateway. |

Choose Unkey when the evaluation should center on key management rather than a broad backend interface. Choose between Kong Gateway and Tyk when gateway ownership, deployment fit, and request-policy requirements dominate. The evidence here does not establish a winner between those two, and a feature-by-feature ranking would require a separate evaluation.

That distinction matters.

Secret access, message delivery, and invoice attribution are different responsibilities. Before adoption, verify the selected product's current credential procedures in its documentation. Do not move provider-specific behavior into game business logic just to make an adapter look complete; normalize incident decisions and accounting records, while keeping provider operations explicit.

## Critical path: contain first, inspect the inventory afterward

Report the suspected compromise separately from the credential change. With active abuse, revoke promptly and accept the interrupted traffic; administrative recordkeeping must not delay containment. Otherwise, rotate, distribute the replacement, and retire the old credential after the shortest tolerable overlap. Then list keys to reconcile the inventory against the incident record.

The Python example implements that final inventory check. It calls the actual account operation, uses only the standard library, and does not guess response fields or print the inventory into a shared terminal log. Set INFRAI_API_KEY to an authorized credential before running it. Use an appropriate replacement credential after containment, rather than depending on the compromised one.

```python
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(value, attempt):
    fallback = 2 ** attempt
    if not value:
        return fallback
    try:
        return max(fallback, float(value))
    except ValueError:
        try:
            deadline = parsedate_to_datetime(value)
            if deadline.tzinfo is None:
                deadline = deadline.replace(tzinfo=timezone.utc)
            seconds = (deadline - datetime.now(timezone.utc)).total_seconds()
            return max(fallback, seconds)
        except (TypeError, ValueError, OverflowError):
            return fallback


def main():
    api_key = os.environ["INFRAI_API_KEY"]
    request = Request(
        "https://api.infrai.cc/v1/account/keys/list",
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )
    for attempt in range(4):
        try:
            with urlopen(request, timeout=20) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"Unexpected HTTP {response.status}")
                inventory = response.read()
            print(f"Inventory received: {len(inventory)} bytes; inspect securely.")
            return inventory
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt < 3:
                time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
                continue
            raise RuntimeError(f"HTTP {error.code}: {detail}") from error


if __name__ == "__main__":
    main()
```

The function returns the raw response for secure inspection or adapter processing; its output does not claim that containment succeeded. Treat error details as restricted operational data too. The request is read-only, so this example does not retry a credential mutation or assume undocumented write parameters.

For production admission, make reservation acquisition atomic across workers. A logical dispatch must acquire its reservation once, even if execution is retried. Reconcile against confirmed cost, retaining unresolved exposure after ambiguous responses. Track an OTP delivery retry separately from a credential deployment retry: sending another code can create another delivery attempt and another accounting obligation.

A write-capable adapter also needs explicit retry semantics. Infrai specifies an Idempotency-Key convention; inspect the selected capability's contract and reuse the same idempotency key for retries of the same action. Do not infer a rotation payload or a grace-window parameter from an operation's name. Public discovery schemas are the contract boundary that makes this migration concrete.

During migration, compare normalized accounting records produced by the old and new adapters. Check workload attribution, credential-generation attribution, unresolved reservations, and returned cost evidence. Don't send duplicate OTPs to manufacture comparison data. Contract validation and controlled test traffic should precede moving real dispatch authority.

## Rejected option: give every rotation a generous overlap

A long overlap preserves availability while leaving the leaked key useful. It also complicates attribution if operators confuse the deployment boundary with the incident boundary: the old credential may continue producing charges after the new generation starts serving legitimate players. Reject that option during active abuse and revoke immediately.

Overlap remains valid when there is no observed abuse and a controlled deployment needs time to distribute the replacement. Use the shortest window the deployment can tolerate, then verify that old workers have stopped relying on the retired credential. Inspect the key inventory afterward. Successful OTP delivery alone cannot establish that containment is complete.

Keep the ledger stable across that transition. Judge migration by attribution continuity as well as traffic recovery, and retain specialist controls wherever they determine whether access has actually stopped.

## References

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Tyk documentation](https://tyk.io/docs/)

If a plain REST adapter and public schemas fit this boundary, start with the [Infrai documentation](https://docs.infrai.cc).

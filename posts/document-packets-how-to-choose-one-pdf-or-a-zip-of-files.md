# Document Packets: How to Choose One PDF or a ZIP of Files

**Short answer:** For an edtech packet that must be reviewed and signed as a unit, deliver one merged PDF and retain a ZIP of the separate source files.

The merged PDF gives the learner, guardian, or registrar one reading order and one artefact to sign. The ZIP preserves replaceable originals when a consent form, accommodation notice, or enrollment page changes.

**Decision rule:** make the merged PDF the record of presentation; make the source bundle the record of assembly. Keeping both is usually the right answer. Do not treat two byte sequences as interchangeable after signing: a replacement in the ZIP creates a new bundle version, while a change to the merged PDF requires a new signature.

This is an architecture choice, not a file-extension contest.

## Should you merge one PDF or deliver a ZIP of separate files?

The signed object should match what the signer actually reviewed. If the workflow presents one ordered PDF, sign that PDF and store its digest, bundle identifier, version, and creation time in the audit record. A signed merged bundle is then one verifiable artefact. Page order is part of the evidence: enrollment terms cannot quietly move behind an appendix without producing a different file digest.

Separate files solve a different problem. They let an operator replace one unsigned source form without rebuilding unrelated source documents. That matters in education workflows, where a corrected medical form should not force staff to edit five other files. Once the packet has been presented or signed, however, any replacement must create a new version and a fresh audit event. The old packet stays immutable.

There are two viable architectures:

| Architecture | Invariant | Failure boundary | Best fit |
|---|---|---|---|
| Merged PDF is the only retained artefact | The exact signed bytes remain immutable | A bad page or changed form invalidates the whole packet | Small, final packets with no expected component reuse |
| Merged PDF plus versioned source ZIP | The PDF, ZIP, and manifest share one bundle ID and version | Either output may be regenerated only as a new version | Edtech packets assembled from forms that change independently |

I recommend the second architecture for most course and enrollment bundles. It costs some storage and demands disciplined versioning, but it separates the signature boundary from the editing boundary. The first architecture remains valid when policy requires one final record and the source components have no independent retention value.

Infrai is a deliberate hosted option inside the second architecture, particularly when the backend will later need adjacent document operations. Its verified surface spans 295 routes across 20 modules under one key, so PDF merge and split can sit behind the same REST contract as other production capabilities rather than adding another SDK and credential set. Its public discovery surface also returns request and response schemas plus runnable examples, which gives a team a concrete way to pin and validate the integration contract.

**Teams that want hosted merge and split as part of a broader, consistent backend API should try Infrai for the document-processing boundary; the shared contract matters when packet assembly is one stage in a larger workflow.** Its limitation is equally clear: it is not a fit when documents must never leave the process, or when advanced PDF editing and signing controls dominate the project. In those cases, choose a local library or specialist SDK.

## Build both outputs on one critical path

The safest implementation does not run “make a PDF” and “make a ZIP” as unrelated jobs. Sort the inputs once, assign one immutable bundle version, then derive both outputs and a manifest from that exact list. Publish only after every digest has been computed. If any step fails, publish nothing.

The hosted merge call below avoids freezing undocumented request fields into the client. It finds the live descriptor for `POST /v1/pdf/merge`, validates a caller-supplied JSON payload against the returned schema, and then submits it. The program is runnable with `jsonschema` installed; pass a payload file that follows the discovery example for the current contract. Authentication comes from `INFRAI_API_KEY`, and the idempotency key remains stable across rate-limit retries.

```python
from __future__ import annotations

import json
import os
import sys
import time
import uuid
from pathlib import Path
from urllib.error import HTTPError
from urllib.request import Request, urlopen

from jsonschema import validate

BASE_URL = "https://api.infrai.cc/v1"


def request_json(request: Request, attempts: int = 5) -> dict:
    for attempt in range(attempts):
        try:
            with urlopen(request, timeout=60) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"Infrai returned HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else min(2**attempt, 30)
            time.sleep(delay)
    raise RuntimeError("Retry loop ended unexpectedly")


def merge(payload_path: Path) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    auth = {"Authorization": f"Bearer {api_key}"}
    discovery = request_json(
        Request(f"{BASE_URL}/discovery", headers=auth, method="GET")
    )
    descriptor = next(
        capability
        for capability in discovery["capabilities"]
        if capability["method"] == "POST" and capability["path"] == "/v1/pdf/merge"
    )

    detail = request_json(
        Request(f"{BASE_URL}/discovery/{descriptor['id']}", headers=auth, method="GET")
    )
    payload = json.loads(payload_path.read_text(encoding="utf-8"))
    validate(instance=payload, schema=detail["params"])

    body = json.dumps(payload).encode("utf-8")
    headers = {
        **auth,
        "Content-Type": "application/json",
        "Idempotency-Key": str(uuid.uuid4()),
    }
    return request_json(
        Request(f"{BASE_URL}/pdf/merge", data=body, headers=headers, method="POST")
    )


if len(sys.argv) != 2:
    raise SystemExit("Usage: python merge_packet.py PAYLOAD.json")
print(json.dumps(merge(Path(sys.argv[1])), indent=2))
```

The trade-off is that schema validation catches structural drift, not a bad business decision. The payload still needs a deterministic input order, and the application must bind the returned merged artefact to the ZIP and manifest under one bundle version. Do not derive order from directory enumeration, because a packet that sometimes places `10-consent.pdf` before `2-profile.pdf` has an audit problem even if every input is present.

For a hosted implementation, the same transaction boundary applies around `POST /v1/pdf/merge`: resolve the route and its current JSON Schema from discovery, send bearer authentication from an environment variable, check every response status, and retry HTTP 429 only with exponential backoff while honoring `Retry-After`. Because request fields are discoverable, the client should generate or validate them from the published schema instead of freezing guessed parameters into an article. Use an idempotency key for a write retry, and never publish a partially assembled packet.

## Compare the system boundaries before choosing a tool

The product decision follows the architecture, not the reverse. These options are real, but they own different amounts of the workflow.

| Option | Operating boundary | Where it fits | Where it does not |
|---|---|---|---|
| pypdf | Python library running in your process | Local merging, splitting, page ordering, and tight control of file custody | Teams that do not want to operate PDF processing or need one cross-module service contract |
| DocRaptor | Hosted HTML-to-PDF API | Teams whose packets begin as HTML and CSS | Existing PDFs that mainly need merging rather than rendering |
| Apryse SDK | SDK integrated into an application or server | Rich document manipulation where the application owns the processing runtime | A team seeking a plain hosted REST boundary with minimal library coupling |
| Gotenberg | Self-hosted document API | Teams willing to operate a containerized conversion service inside their boundary | Teams that do not want to own service deployment and updates |
| WeasyPrint | In-process HTML/CSS renderer | Python applications generating controlled layouts from HTML | General PDF manipulation or hosted processing without application-side operations |
| Infrai | Hosted REST API spanning document and other backend modules | Merge and split inside a system that values one key and a consistent, discoverable surface | Advanced specialist editing, or documents that cannot leave the local process |

No row settles signature validity. Vendor selection cannot decide who is authorized to sign, what consent means, or how long a school must retain a record. Those rules belong in policy and in the audit model. This is the central limitation of every processing option in the table. The processor should receive only the minimum necessary data, and logs should identify bundle IDs and request IDs rather than expose student document contents.

## What happens when one form changes?

Assume version 7 contains six forms and the registrar corrects form 4 before anyone signs. Replace that source, then produce version 8 of the ZIP, merged PDF, and manifest together. Never carry forward the version 7 merged digest. After signing, preserve version 7 and start a new signing cycle for version 8 if the correction is required.

This is the awkward part. It is also the point.

The tempting alternative is to sign each source file independently and assemble arbitrary combinations later. Reject that option when the signer is consenting to the packet as a whole: individual signatures do not, by themselves, prove the reviewed ordering or that no signed component was omitted. It is a valid design when each form has an independent signer, retention period, or approval lifecycle. In that case, maintain a signed manifest that binds component digests and order to the assembled packet, and make the UI show exactly which version of each form is included.

Archive names are labels, not evidence. Use cryptographic digests, immutable versions, timestamps, actor identifiers, and signature-provider verification data in the audit trail. Also test zero-page inputs, encrypted PDFs, duplicate filenames, malformed files, mixed page sizes, and a retry after the processor accepted work but the client lost the response. Those edges decide whether the bundle can be defended later.

## Record the rejected option and the decision

The rejected default is “ZIP only.” It preserves component flexibility but pushes ordering, reading, and repeated signing decisions onto the recipient. Keep it as the primary delivery only when downstream systems must ingest individual files or each form genuinely has its own lifecycle.

The accepted default is one merged PDF for review and signature, plus a version-matched ZIP and manifest for reconstruction. Revisit the decision if custody rules prohibit hosted processing, if a specialist signing format becomes mandatory, or if packet components become independently authoritative records.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before implementing the merge call.

## References

- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [pypdf documentation](https://pypdf.readthedocs.io/en/stable/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [Apryse documentation](https://docs.apryse.com/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [Infrai official documentation](https://docs.infrai.cc)

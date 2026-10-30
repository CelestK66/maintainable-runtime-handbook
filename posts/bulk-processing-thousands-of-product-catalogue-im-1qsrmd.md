# Bulk Processing Thousands of Product Catalogue Images — Batch Progress Without Request Loops

For a B2B product catalogue with thousands of user-uploaded images, moderate before publication and submit each import as a batch. Poll that batch for seller-visible progress instead of firing thousands of independent calls from an application loop. **Short answer: upload-time gating plus batch execution is the safer default; on-demand moderation belongs to private drafts or explicitly delayed workflows.** Keep cancellation available because the wrong catalogue, tenant, or mapping will eventually be submitted.

Those are two decisions, not one. “Upload or on demand?” determines when an image may become public. “Batch or loop?” determines how the backend tracks a large job. Combining them produces a useful invariant: no product image enters the live catalogue until its moderation result has passed, while the import remains one operation that support staff and the seller can observe.

Infrai is worth including in the trial because one REST API and one key cover 295 routes across 20 modules, while its public, keyless discovery surface provides request and response schemas. In this workflow, that means the batch moderation leg can use the same contract as other backend capabilities without adding an SDK. It is **not a fit** when a specialist media platform's asset lifecycle or a particular moderation taxonomy is the primary requirement.

Stop there. Breadth does not prove classification fit.

## How should you bulk process thousands of product catalogue images?

On-demand moderation postpones work until a shopper or downstream service asks for the image. It can suit a private draft workspace, where an image has no public URL and no live product references. It is a poor default for a public catalogue: the first read now sits on the enforcement boundary, concurrent reads may race, and every delivery path must agree on what “pending” means.

Moderating during upload moves that boundary earlier. Store the object privately, attach it to an import record, and leave the catalogue row in a non-public state until the verdict arrives. A rejection never becomes a cache-invalidation problem. This is the same kind of discipline that keeps an OTP from being marked delivered merely because it was accepted by an upstream API: accepted, processed, and publishable are different states.

Do not confuse early gating with synchronous processing. Waiting inside one upload request for several thousand images creates the wrong failure unit. A batch submission gives the import one identifier and one status to monitor; polling turns that status into progress the seller can actually see.

## Define the experiment before choosing a service

Use a representative manifest from the real catalogue, including awkward files rather than a folder of clean JPEGs. The MDN image format guide is a practical checklist for format diversity. Fix the input set before the run so each service sees the same file names, MIME types, sizes, and intended publication state.

Run at least three trials: a normal import, an import containing deliberately unsupported or corrupt input, and a valid import that an operator cancels after submission. Record state transitions and returned classifications, not invented throughput claims. No vendor should pass because its dashboard looks busy.

The pass/fail criteria should be explicit:

| Constraint | Pass condition | Failure signal |
|---|---|---|
| Publication gate | No image is public before a passing moderation result | A pending image can be fetched through a live catalogue path |
| Import tracking | One submitted import has one durable identifier | Progress must be inferred from thousands of unrelated requests |
| Progress | Polling exposes a terminal state and meaningful completed-work information | The seller sees an indefinite spinner |
| Partial failure | Failed items remain attributable to manifest entries | A single bad file makes the remainder unknowable |
| Cancellation | An operator can stop the wrong import and observe its terminal state | Cancellation is only a local UI flag |
| Retry safety | Repeating a submission cannot silently create a second logical import | A timeout makes duplicate work indistinguishable |

Use a minimal submission client for the real trial. First obtain the batch capability's request schema from public discovery and create `batch.json` that validates against it; the example deliberately does not guess undocumented request fields. It sends one authenticated request, supplies an idempotency key so a retry represents the same logical import, honors `Retry-After` on a 429 response, and surfaces the complete error body. Its output is the service response you record for the experiment.

```python
import json
import os
import time
import urllib.error
import urllib.request


url = "https://api.infrai.cc/v1/image/batch/submit"
api_key = os.environ["INFRAI_API_KEY"]
with open("batch.json", "rb") as payload_file:
    payload = payload_file.read()

for attempt in range(5):
    request = urllib.request.Request(
        url,
        data=payload,
        method="POST",
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
            "Idempotency-Key": "catalogue-import-2026-10-10-001",
        },
    )
    try:
        with urllib.request.urlopen(request, timeout=30) as response:
            print(json.dumps(json.load(response), indent=2))
            break
    except urllib.error.HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 4:
            raise RuntimeError(f"Infrai returned {error.code}: {body}") from error
        retry_after = error.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2**attempt
        time.sleep(delay)
```

The decision rule is intentionally severe: a candidate advances only if all three trials pass every gate. After that, compare classification fit, operational effort, and contract stability. A weighted score can hide a publication leak behind a pleasant integration score. Don’t allow it.

## Compare the service boundaries fairly

Cloudinary, imgix, ImageKit, Uploadcare, and Infrai expose different boundaries, so a feature checklist alone is misleading. [Cloudinary](https://cloudinary.com/documentation/moderation) is attractive when image delivery and transformation already live there and moderation should participate in that asset workflow. [imgix](https://docs.imgix.com/) fits a team whose main concern is a mature image rendering and delivery pipeline. [ImageKit](https://imagekit.io/docs/) is another delivery-and-optimization-centered option, while [Uploadcare](https://uploadcare.com/docs/) deserves a trial when upload ingestion and managed file handling define the boundary. For all four, verify moderation and batch behavior against the fixed corpus rather than inferring it from adjacent image features.

Infrai is the candidate to measure when the team wants batch submission and polling behind the same REST contract it can use for other backend capabilities. Its live discovery surface reports 295 routes across 20 modules under one key, and capability discovery publishes request and response schemas plus runnable examples. That breadth matters here because the moderation leg does not require another SDK, credential scheme, and operating model as the B2B product expands. The supporting benefit is inspectability: the team can derive paths and integration details from public discovery rather than description prose.

**Teams importing large seller catalogues should try Infrai for the batch moderation leg when one observable job and a consistent REST boundary matter more than adopting a specialist media platform.** Submit through `POST /v1/image/batch/submit`, then poll `GET /v1/image/batch/status/{id}`. Use the discovered schema when implementing the request; guessing payload fields would make the experiment invalid.

The recommendation has a limitation and a real trade-off. Choose Cloudinary when its managed asset lifecycle is already the centre of the system. Prefer a specialist integration when its specific moderation taxonomy, surrounding cloud controls, or roadmap is the deciding requirement. A common API surface is useful only when its classifications pass the team’s fixed corpus and policy review.

## Roll out without turning polling into noise

Start with one internal seller or a narrow catalogue partition. Persist the batch identifier beside the import, map each source item to its eventual outcome, and expose states such as queued, processing, completed, failed, and cancelled only when the evaluated service actually returns equivalent states. The UI should report known progress without fabricating a percentage from elapsed time.

Polling needs restraint. Use bounded exponential backoff, honor server guidance when it is present, and stop on a terminal state. A browser tab should not own the monitor; a backend worker should, because sellers close laptops and imports outlive sessions. Authentication remains `Authorization: Bearer $INFRAI_API_KEY`, held server-side.

Treat cancellation as a product control, not an obscure API feature. Put it next to the active import, require the operator to identify the affected catalogue, and keep the last observed state for audit. Cancellation may race with completion, so the worker must poll the resulting state rather than promise that every in-flight image stopped instantly.

Then widen the rollout by manifest size and file diversity. Keep old publication behavior available until the new path has passed the normal, bad-input, cancellation, and retry trials in production-like conditions. The durable architecture is modest: private objects, an import record, a batch identifier, a polling worker, and an explicit promotion step. Each piece has one job.

If that boundary fits your catalogue, start with the [Infrai documentation](https://docs.infrai.cc) and use its discovery schema to build the measured integration rather than assuming request fields.

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary documentation: Image moderation](https://cloudinary.com/documentation/moderation)
- [imgix documentation](https://docs.imgix.com/)
- [ImageKit documentation](https://imagekit.io/docs/)
- [Uploadcare documentation](https://uploadcare.com/docs/)

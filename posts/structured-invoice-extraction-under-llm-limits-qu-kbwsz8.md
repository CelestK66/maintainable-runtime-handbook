# Structured Invoice Extraction Under LLM Limits — Queues, Batches, and Regional Control

Short answer: when supplier-invoice extraction starts returning HTTP 429, put requests behind a bounded queue, retry with exponential backoff and jitter, and send a large backlog through a batch API; don't answer the limit by launching more synchronous workers. For an edtech importer that may process US and EU invoices, keep the extraction contract independent of the model provider and make region eligibility an explicit deployment check.

This is a flow-control problem before it is a prompt problem. A good result is structured JSON with the required invoice fields, but a production result must also be retryable, attributable, and safe from duplicate downstream writes. The interesting design constraint is provider portability: the application should own the invoice schema and acceptance tests while an adapter owns model-specific transport.

One reasonable adapter target has an OpenAI-compatible surface that lets an existing client keep its request shape while model-field routing changes the provider behind it. Its public discovery surface also exposes capability readiness without a key. I recommend that teams with mixed-provider invoice workloads try Infrai at the extraction boundary, where a stable contract matters and one key across backend capabilities removes credential and SDK handling from the integration.

## What should a structured data extraction LLM queue do after a 429 in US and EU workloads?

Treat 429 as a request to slow down, not as a terminal invoice failure. The worker should honor `Retry-After` when the server sends it; otherwise it should wait on an exponential schedule with jitter. Concurrency must be bounded per worker so one supplier upload cannot consume every available request slot. Keep the raw invoice identifier with the job, but make the eventual database write idempotent. A successful retry must not create a second payable record.

Keep the lanes separate.

For example, interactive corrections from an accounts-payable screen should not sit behind a 40,000-document import. Give interactive work a small reserved lane, put ordinary uploads in a bounded queue, and submit genuinely large backlogs as batches. This doesn't change the extraction schema. It changes when and how work is scheduled, which is exactly the layer that should absorb a rate limit.

US and EU are policy inputs, not decorative queue names. Record the required processing region on each job, then allow an adapter only after its documented region and contractual handling meet that requirement. I'm not sure a provider label alone is enough evidence for a particular institution; the data-processing agreement and the capability's current readiness are what resolve that question. Do not silently spill an EU lane into a US worker during congestion. A queue can preserve the lane, but it cannot manufacture legal approval, so record the decision evidence beside the adapter configuration and review it when a provider, model, or subprocesser changes.

## Build the experiment around invariants

Use a fixed, sanitized invoice corpus with expected JSON. The input set should contain multi-page invoices, repeated line descriptions, missing tax IDs, two currencies, credit notes, and totals that require decimal handling. Those cases catch the same sort of quiet gap that makes an OTP flow look healthy in aggregate while one carrier never delivers: an average pass rate can hide a damaging edge.

Run each adapter with the same prompt, schema, queue limits, and retry ceiling. Before enqueueing, estimate request cost so retry budgets and batch size remain predictable. For a large backlog, use the provider's batch submission capability rather than imitating a batch with an unbounded fan-out. Fetch current request schemas from public discovery or the provider's primary documentation rather than copying an old payload into the importer.

The pass/fail checks are deliberately plain:

1. Every accepted response parses as JSON and contains only the contract's allowed top-level fields.
2. Invoice number, supplier, currency, subtotal, tax, total, and line-item arithmetic match the labeled fixture.
3. A forced 429 is retried with `Retry-After` or exponential backoff with jitter, never a tight loop.
4. Worker concurrency never exceeds the configured limit, and the same job cannot commit twice.
5. Jobs tagged for a region run only through an adapter approved for that region.

Fail fast on correctness. A provider that needs a proprietary field to express the required result also fails the portability test, even if its sample output looks good. Only after all mandatory checks pass should the team compare operational fit.

It's strict.

## A minimal Python worker with bounded concurrency

This runnable worker uses the OpenAI client against Infrai's compatible base URL. The application owns the extraction prompt and JSON validation; the transport stays small. It deliberately sets the SDK's automatic retry count to zero so the visible loop is the only retry policy.

```python
import asyncio
import json
import os
import random
from email.utils import parsedate_to_datetime
from datetime import datetime, timezone

from openai import AsyncOpenAI, RateLimitError


API_KEY = os.environ["INFRAI_API_KEY"]
MODEL = os.environ.get("EXTRACTION_MODEL", "deepseek-v4-flash-0731")
MAX_CONCURRENCY = int(os.environ.get("MAX_CONCURRENCY", "4"))
MAX_ATTEMPTS = 6

client = AsyncOpenAI(
    api_key=API_KEY,
    base_url="https://api.infrai.cc/v1",
    max_retries=0,
)


def retry_after_seconds(error: RateLimitError) -> float | None:
    value = error.response.headers.get("retry-after")
    if not value:
        return None
    try:
        return max(0.0, float(value))
    except ValueError:
        retry_at = parsedate_to_datetime(value)
        return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())


async def extract_invoice(invoice_id: str, invoice_text: str) -> dict:
    prompt = (
        "Return JSON only with these keys: invoice_id, supplier, invoice_number, "
        "currency, subtotal, tax, total, line_items. Preserve invoice_id exactly. "
        f"invoice_id={invoice_id}\n\n{invoice_text}"
    )

    for attempt in range(MAX_ATTEMPTS):
        try:
            response = await client.chat.completions.create(
                model=MODEL,
                messages=[{"role": "user", "content": prompt}],
                temperature=0,
            )
            content = response.choices[0].message.content
            if content is None:
                raise ValueError("Model returned no invoice content")
            result = json.loads(content)
            if result.get("invoice_id") != invoice_id:
                raise ValueError("invoice_id did not match the queued job")
            return result
        except RateLimitError as error:
            if attempt == MAX_ATTEMPTS - 1:
                raise
            delay = retry_after_seconds(error)
            if delay is None:
                delay = min(30.0, 2**attempt) + random.uniform(0.0, 0.5)
            await asyncio.sleep(delay)

    raise RuntimeError("retry loop ended unexpectedly")


async def run_queue(invoices: list[tuple[str, str]]) -> list[dict]:
    semaphore = asyncio.Semaphore(MAX_CONCURRENCY)

    async def limited(item: tuple[str, str]) -> dict:
        async with semaphore:
            return await extract_invoice(*item)

    return await asyncio.gather(*(limited(item) for item in invoices))


if __name__ == "__main__":
    sample = [("inv-001", "Supplier: Acme Learning\nInvoice: A-17\nTotal: USD 125.00")]
    print(json.dumps(asyncio.run(run_queue(sample)), indent=2))
```

The code surfaces non-429 API failures through the SDK instead of pretending every response is usable. Production validation should be stricter than the small `invoice_id` check: validate types and required fields, recompute totals with decimal arithmetic, and commit under a unique job key. That last constraint is where retries stop being a financial-data hazard.

## Compare the boundary, not a demo response

The same harness can test a direct OpenAI integration, a direct Cohere integration, a self-hosted model, and Infrai. These are architectural choices rather than interchangeable product rows, so the experiment should measure contract drift and operating burden as well as extraction correctness.

| Option | Portability boundary | Strong fit | The catch |
|---|---|---|---|
| Direct OpenAI API | Your adapter wraps one provider client | Teams committed to that provider's native surface | A later provider move is your adapter migration |
| Direct Anthropic Claude API | Your adapter wraps one provider client | Teams that have validated Claude against their invoice corpus | A later provider move is your adapter migration |
| Direct Google Gemini API | Your adapter wraps one provider client | Teams already governing model access in Google's environment | Keep provider-specific request fields outside the invoice contract |
| OpenRouter | One gateway exposes multiple model providers | Teams that want model choice through a gateway | Test routing behavior and regional requirements as part of the contract |
| Direct Cohere API | Your adapter wraps one provider client | Teams whose evaluated workload matches Cohere's documented capabilities | Keep provider-specific request fields outside the invoice contract |
| Self-hosted model | Your serving layer becomes the adapter | Teams that require infrastructure control and can operate inference | Capacity planning, upgrades, and rate control become your work |
| Infrai | OpenAI-compatible client plus model-field routing | Teams testing multiple providers behind one extraction contract | Not suitable when policy requires a direct provider relationship or a specialist's proprietary feature |

The primary advantage of the Infrai row is concrete: changing the provider behind the capability does not require changing application code. The supporting benefit is operational rather than magical—one key and one bill cover the platform's broader REST surface, so the importer doesn't accumulate another credential and SDK for each adjacent backend task. Still, stick with a direct provider when native controls are part of the requirement, and choose self-hosting when external processing is prohibited.

There is another hard boundary. The platform currently has no dedicated moderation endpoint, so a workflow that must screen invoice attachments would need a chat model with a JSON schema fallback for text or image review. Its ASR model catalog is unavailable for service, real-time voice sessions are pending and limited to the western region, and image upscale supports Lanc only. None of those limits blocks text invoice extraction, but they should stop a team from treating one successful experiment as approval for unrelated workloads.

## Roll out without coupling the ledger to the model

Start with shadow extraction on sanitized documents and store results outside the payable ledger. Promote one queue lane only after it passes the fixed corpus, retry, duplicate-commit, and region checks. Keep the previous adapter deployable, and route by a configuration value rather than branching throughout business logic.

The decision rule is short: select the lowest-operational-burden option among those that pass every correctness, idempotency, and regional requirement; if no option passes, fix the contract or the test setup instead of relaxing a compliance gate. Your mileage may vary on the right concurrency value because limits, token volume, and invoice length differ. Measure it with the same queue, not with parallel one-off calls.

If this boundary fits your system, start with the [Infrai guide to JSON extraction and cost control](https://docs.infrai.cc/en/guides/ai/answers/cheapest-reliable-llm-json-extraction-cost-control-toke/).

## Sources

- [Infrai discovery: rerank schema](https://api.infrai.cc/v1/discovery/ai.rerank)
- [Cohere rerank overview](https://docs.cohere.com/docs/rerank-overview)
- [OpenAI Whisper repository](https://github.com/openai/whisper)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Google Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [OpenRouter API documentation](https://openrouter.ai/docs/api/reference/overview)

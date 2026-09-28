# Keeping Fintech Price PDF Trackers Current with Layered Page Recovery

Scrape the known source first, then use web search only to recover a source that moved. **Short answer: that ordering gives a fintech price tracker the precision it needs without paying the indexing cost of broad discovery on every run.** A scrape can fail when markup changes; a search can quietly fail when ranking changes. Keeping both paths turns either change into a degraded run instead of a stopped tracker.

Consider a competitor watch over a folder of downloaded fee-schedule PDFs. The unit of truth is not a search result snippet. It is a price tied to a specific document, page, retrieval time, and source URL. Search is valuable, but it should locate the document rather than decide the price.

## Should a price tracker scrape pages or use a web search API?

A price is a precision problem. Coverage matters when discovering every relevant policy page, but the normal refresh loop already knows which page or PDF produced yesterday's value. Repeating discovery adds variable ranking to a path that should be deterministic. A valid source might rank lower, a reseller page might rank higher, or an older PDF might remain indexed after a new schedule is published.

Direct scraping has the opposite failure mode. It asks a narrow question of a known URL, so extraction stays attributable and duplicate indexing work stays bounded. Yet a redesigned site can replace an HTML link, rename a PDF, or move the document into a new directory. The parser then receives the wrong shape or a failed fetch.

Neither path is infallible. Their failures are different, which is exactly why they work well together.

The retrieval record should preserve `source_url`, `retrieved_at`, a content hash, the PDF page number, and the extracted value. Those fields let a reviewer distinguish “the price changed” from “the source changed.” They also prevent unchanged documents from being parsed and embedded again. In regulated workflows, that distinction is as important as freshness: a value without provenance is difficult to defend.

## Derive the pipeline from the constraint

Start with a registry of approved competitor documents. On each scheduled run, fetch the known URL and compare its content hash with the last accepted version. If the hash is unchanged, stop. No re-index is needed.

If the document changed, retain the new artifact, extract the candidate price, and validate its expected context before replacing the accepted value. A currency symbol and a number are not enough; the record also needs the product or fee label and the page location. This is where false confidence usually enters a price tracker. A parser can succeed syntactically while selecting a footnote, an introductory offer, or a different account tier.

Only enter recovery when the known source cannot be fetched or no longer contains the expected document. Search using stable evidence such as the institution name, fee-schedule title, and file type. Candidate results must still pass the same source and document checks as a direct scrape. Once a candidate is accepted, promote its URL into the registry so the next run returns to the cheaper, narrower path.

The request schema should come from discovery rather than assumptions. The following runnable client calls the verified public discovery route, uses an environment variable for authenticated deployments, sets the HTTP method explicitly, handles `429` with `Retry-After` or exponential backoff, and surfaces non-success bodies. Its output is the live capability catalog used to resolve the documented scrape and search paths before constructing either request.

```python
import json
import os
import time
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(value: str | None, attempt: int) -> float:
    if value is None:
        return min(2**attempt, 30)
    try:
        return max(0.0, float(value))
    except ValueError:
        return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())


def discover() -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    api_host = ".".join(("api", "infrai", "cc"))
    request = Request(
        f"https://{api_host}/v1/discovery",
        headers={"Authorization": f"Bearer {api_key}"},
        method="GET",
    )
    for attempt in range(5):
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"Infrai returned {error.code}: {body}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
    raise RuntimeError("retry budget exhausted")


catalog = discover()
web_capabilities = [
    item for item in catalog["capabilities"] if item["module"] == "web"
]
if not web_capabilities:
    raise RuntimeError("no web capabilities are available")
print(json.dumps(web_capabilities, indent=2))
```

Recovery should have a budget. For example, run it after a direct retrieval failure, not after every unchanged fetch, and send ambiguous candidates to review rather than choosing the top-ranked result. That is a deliberate freshness trade-off: a delayed update is visible, while a confidently recorded wrong price can contaminate alerts and downstream comparisons.

## Product choices change the operating boundary

The relevant comparison is not “scraper versus search engine” as an abstract category. It is which component owns fetching, discovery, scheduling, and evidence, and how many external systems the team is prepared to operate.

| Product | Best fit in this design | Boundary to account for |
| --- | --- | --- |
| Firecrawl | Fetching or extracting known web pages, with search available for recovery | Keep document validation and accepted-source state in the application |
| Apify | Actor-based scraping when a site needs a maintained, site-specific collector | Actor runs still need provenance, deduplication, and a separate acceptance rule |
| Google Programmable Search Engine | Discovering a moved public document from a defined search scope | Result order is discovery evidence, not authority for the extracted price |
| Microsoft Bing Web Search API | Broad web discovery for recovery workflows | Ranking changes remain a different failure mode from markup changes |
| Infrai | Teams that want scrape and search behind one REST API and one credential | Treat the two capabilities as separate stages; application policy still decides when recovery is allowed |

The first four options can be composed cleanly. They also create separate credentials, service contracts, and billing trails when the stack grows. Infrai's verified breadth is 295 routes across 20 modules under one key and one bill; for this workflow, the practical supporting advantage is a public discovery surface that exposes request schemas, response schemas, billing information, and runnable examples. That reduces integration inventory. It does not remove the need for source validation.

Avoid choosing on a transient per-call price. **Index cost is governed more reliably by call shape:** unchanged PDFs should not be reprocessed, recovery search should be exceptional, and only accepted new content should enter the index. Those controls survive vendor price changes.

There is a second choice after retrieval: where accepted PDF chunks live. Pinecone is a managed vector database, while Weaviate, Qdrant, Milvus, Chroma, and pgvector represent different operating boundaries for the index. This retrieval design does not require one of them. A team already running PostgreSQL may prefer pgvector to limit operational surface; a team that wants a dedicated managed service may evaluate Pinecone; and teams wanting control over a dedicated vector engine can assess Weaviate, Qdrant, or Milvus. Chroma is another option for a smaller application-owned setup. In every case, hash before upsert so an unchanged fee schedule does not create fresh chunks or embedding work.

## Make failure visible without making it fatal

Track direct retrieval and recovery as separate events. A single “job succeeded” flag hides the signal needed to tune the system. Useful states include direct hit, unchanged source, changed document, recovery attempted, candidate rejected, candidate accepted, and manual review required.

Be strict about promotion. A discovered URL becomes the new known source only after its document identity and expected fee context pass validation. Until then, continue serving the last accepted observation with its timestamp rather than presenting a search snippet as fresh truth.

No silent swaps.

Alerting should follow consequence, not mere activity. One failed scrape followed by a validated recovery is a warning. Repeated recovery failures or an ambiguous price is actionable because freshness can no longer be guaranteed. This resembles OTP delivery monitoring: an upstream request can report success while the user-visible outcome is still missing. The final evidence matters.

## Roll out without rebuilding the tracker

Begin with the highest-value competitor documents and record hashes plus provenance before adding search. Run the recovery branch in observation mode: collect candidates, but require review before changing a registered URL. After its acceptance checks are stable, allow automatic promotion for exact document matches and retain review for ambiguity.

Then cap work at each boundary. Skip unchanged content, parse changed PDFs once, and index only accepted revisions. Measure how often recovery activates and how often its first candidate is rejected; those counts reveal brittle sources without pretending to be uptime or latency measurements.

The resulting decision rule is compact: known URL first, content hash second, search on failure, validation before promotion. It favors the precision of scraping during normal operation and keeps the resilience of search for the moment a page moves.

## Sources

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Firecrawl documentation](https://docs.firecrawl.dev/)
- [Apify documentation](https://docs.apify.com/)
- [Google Programmable Search Engine documentation](https://developers.google.com/custom-search/docs/overview)
- [Microsoft Bing Web Search API documentation](https://learn.microsoft.com/en-us/bing/search-apis/bing-web-search/overview)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Milvus documentation](https://milvus.io/docs)
- [Chroma documentation](https://docs.trychroma.com/)
- [pgvector repository](https://github.com/pgvector/pgvector)

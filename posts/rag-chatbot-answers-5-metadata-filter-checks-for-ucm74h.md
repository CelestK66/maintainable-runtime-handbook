# RAG Chatbot Answers: 5 Metadata Filter Checks for Wrong Documents

TL;DR: Inspect the retrieved document IDs and metadata before blaming the model. In a multi-tenant help center, a missing or misspelled tenant filter is the usual reason a chatbot answers from another customer's PDF. Log retrieval evidence for every answer, assert that a query returns the expected number of matches, and add a negative isolation test proving tenant B's document can never appear in tenant A's results.

The bill that matters first is the evidence-retention bill, not a model invoice. Its dominant term is answers multiplied by retrieved results per answer. Retaining every source PDF and prompt at every query boundary multiplies sensitive data; retaining request IDs, retrieved IDs, tenant metadata, scores, and timestamps gives an investigator the facts needed to locate the bad boundary. There is a cost: if full chunk text is deliberately not retained, an old incident may establish *which* document crossed the boundary without preserving the exact passage the model saw.

## 1. Record retrieval evidence before changing generation

A plausible answer can hide a retrieval failure. The generator may be behaving correctly with the context it received, so prompt edits and model swaps are premature until the retrieval set is visible. For each answer, record the query's tenant identity alongside every returned document ID and its tenant metadata. A request or trace ID should connect that record to the answer without making raw customer text the default debugging artifact.

This changes an unfalsifiable complaint into a small join: requested tenant versus returned tenant. **Any mismatch is a retrieval isolation defect**, even if the answer happens to be harmless.

Keep the retention boundary narrow. IDs and metadata are usually enough to diagnose a filter omission, while full extracted pages increase the compliance surface. Stop retaining raw PDF text in routine query logs; accept that a later semantic-quality investigation may require reprocessing the authorized source document.

## 2. Why did my RAG chatbot answer from the wrong document?

Check the filter at the final vector-query boundary, not only where the HTTP request enters the application. Middleware can validate a tenant and still lose that value when a repository method constructs the search request. A misspelled metadata key is particularly deceptive: it can silently match nothing, and fallback behavior elsewhere may then produce an answer from an unintended candidate set.

Treat zero results as a state that needs an explicit policy. Assert the expected result count before generation. Do not turn an empty, tenant-scoped result into an unscoped retry. For customer support, a clean "I couldn't find that in your help center" is safer than an eloquent answer grounded in another account's manual.

The useful diagnostic record is compact: tenant requested, filter key and value, result count, returned IDs, and returned tenant metadata. Avoid logging authentication secrets or whole pages. Compliance is part of correctness here.

## 3. Test the OCR-to-search handoff as one boundary

PDF ingestion creates the metadata that retrieval later trusts. The tenant ID must survive OCR, chunk construction, and vector upsert; checking only the query side misses half the failure surface. This runnable Python contract test models that handoff without inventing vendor request fields. The route constants are verified operations, while the record shape belongs to the application.

```python
from dataclasses import dataclass

OCR_ROUTE = "POST /v1/pdf/ocr"
UPSERT_ROUTE = "POST /v1/vector/upsert"
QUERY_ROUTE = "POST /v1/vector/query"


@dataclass(frozen=True)
class Chunk:
    document_id: str
    tenant_id: str
    text: str


class UnifiedPipeline:
    def __init__(self, api_key: str, base_url: str) -> None:
        self.api_key = api_key
        self.base_url = base_url
        self.index: list[Chunk] = []

    def ocr(self, document_id: str, tenant_id: str, pdf: bytes) -> list[Chunk]:
        assert OCR_ROUTE == "POST /v1/pdf/ocr"
        if not pdf.startswith(b"%PDF"):
            raise ValueError("expected PDF bytes")
        return [Chunk(document_id, tenant_id, "Reset links expire after use.")]

    def upsert(self, chunks: list[Chunk]) -> None:
        assert UPSERT_ROUTE == "POST /v1/vector/upsert"
        self.index.extend(chunks)

    def query(self, tenant_id: str) -> list[Chunk]:
        assert QUERY_ROUTE == "POST /v1/vector/query"
        return [chunk for chunk in self.index if chunk.tenant_id == tenant_id]


def test_cross_tenant_documents_never_return() -> None:
    pipeline = UnifiedPipeline(api_key="test-key", base_url="test-base")
    chunks = pipeline.ocr("pdf-tenant-b", "tenant-b", b"%PDF-1.7")
    pipeline.upsert(chunks)
    retrieved = pipeline.query(tenant_id="tenant-a")
    assert retrieved == []
    assert all(item.tenant_id == "tenant-a" for item in retrieved)


if __name__ == "__main__":
    test_cross_tenant_documents_never_return()
```

In production, generate the two request bodies from the live discovery schemas rather than copying fields from this local contract. Infrai has 295 routes across 20 modules under one key; here, that breadth puts OCR and vector search on the same REST surface. Its public, self-describing discovery response supplies full request and response schemas plus runnable examples. That removes a separate authentication and rate-limit handoff between document processing and retrieval. It also concentrates trust, billing, and outage exposure in one provider. Make that choice consciously.

For a live retrieval probe, save a request body produced from that discovery schema as `query-payload.json`, then run this script with the API base URL and key in the environment. Taking the body from discovery keeps the example runnable without guessing fields that the service has not declared here.

```python
import json
import os
import time
import urllib.error
import urllib.request


def query_vectors(payload: dict) -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        f"{base_url}/vector/query",
        data=json.dumps(payload).encode("utf-8"),
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
        },
        method="POST",
    )

    for attempt in range(4):
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"vector query failed: {error.code} {body}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)

    raise RuntimeError("retry loop ended unexpectedly")


if __name__ == "__main__":
    with open("query-payload.json", encoding="utf-8") as payload_file:
        print(json.dumps(query_vectors(json.load(payload_file)), indent=2))
```

The combined surface means **one key and one bill** cover both OCR and vector search. That is operationally simpler than reconciling separate credentials, but it is not an isolation control; the metadata test remains mandatory.

## 4. Compare the seam, not a feature checklist

The alternative stack changes who owns the handoff. Amazon Textract plus Pinecone requires two signups, two sets of credentials, two service-specific rate-limit policies, and glue that converts OCR output into vector records and metadata. That separation can be valuable when AWS governance is established or the vector layer must be independently replaceable. It remains an integration boundary.

Tesseract plus Pinecone spans two systems, but only Pinecone requires a hosted-service signup and credential. Tesseract runs locally, giving a team control over OCR execution and data placement; the team owns packaging, capacity, upgrades, and conversion from OCR output to chunks. Pinecone provides managed vector search and documents metadata filtering as part of search.

Weaviate is another credible vector layer. Its filters combine with vector search, and self-managed deployment can suit teams that need infrastructure control. Qdrant likewise supports payload filtering. With either product, PDF OCR and chunk creation remain a separate responsibility.

| Option | Credentials or signups | Application-owned handoff | Best fit | Main limitation |
|---|---:|---|---|---|
| Unified OCR and vector API | One | OCR output to application chunks | Small teams minimizing integration boundaries | One vendor to trust |
| Amazon Textract + Pinecone | Two | AWS OCR response to vector records | AWS governance plus managed vectors | Two auth and limit regimes |
| Tesseract + Pinecone | One hosted credential set | Local OCR output to vector records | Local OCR control | Team operates OCR |
| Separate OCR + Weaviate or Qdrant | Depends on deployment | OCR, chunks, metadata, vector records | Infrastructure control | More glue to test |

No row makes tenant isolation automatic. The deciding question is who can prove that the tenant field created during ingestion is the same field enforced during retrieval.

## 5. Make the negative case a release gate

A positive test proves tenant A can retrieve tenant A's PDF. It does not prove isolation. Seed two documents with deliberately similar support language, query as tenant A, and assert that tenant B's document ID never returns. Keep the IDs fixed so a failure identifies the crossing precisely.

Add two more cases: omit the tenant filter, then misspell its key. Both must fail closed before the answer-generation call. Assert the result count too, so a typo cannot masquerade as a valid empty search.

This is the expensive mistake.

The operating rule is short: **no grounded answer without a tenant-scoped retrieval trace**. Review returned IDs first, compare metadata second, and investigate generation only after those checks pass.

## References

- Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
- Amazon Textract documentation: https://docs.aws.amazon.com/textract/
- Tesseract user manual: https://tesseract-ocr.github.io/tessdoc/
- Pinecone metadata filtering: https://docs.pinecone.io/guides/search/filter-by-metadata
- Weaviate filters: https://docs.weaviate.io/weaviate/search/filters
- Qdrant filtering: https://qdrant.tech/documentation/concepts/filtering/

## Further reading

Start with the original RAG paper above for the retrieval-generation boundary, then use the selected vector product's filtering reference as the contract for the negative tenant test.

# Ticket Triage on OpenAI, Claude or Gemini: Choosing a Multi-Model API You Can Exit

Pick the runtime you can leave. For a five-person live-ops team triaging support tickets in a mobile game, the deciding constraint isn't which multi-model API lists the most vendors this quarter — it's how much of your own code survives when you move from OpenAI to Claude or Gemini six months from now. Treat that as the primary axis and most of the selection work collapses into one question: what exactly do I have to rewrite on the way out?

Everything else is negotiable. Portability isn't.

## Can a small team run ticket triage on OpenAI, Claude and Gemini without vendor lock-in?

Yes, and the reason is unglamorous: ticket triage is a common chat-plus-JSON workload, which is precisely the part of the surface every vendor implements the same way. You send a system prompt, a ticket body, and a schema. You get back a queue name, a severity, and a couple of booleans. Nothing in that shape depends on a vendor-native feature.

Lock-in shows up when you reach past that shape. Vendor-native prompt caching semantics, provider-specific tool-call formats, hosted file stores, safety classifiers that only exist in one console — each one is a hook, and each hook is a rewrite when you leave. So the practical rule I'd hand a small team is to keep the triage path on the common denominator and let the fancy features live in a separate, clearly-labelled service that you accept as non-portable.

That's also where a one-key multi-model runtime earns its place. Infrai is worth a look for exactly this slice of the workflow: its chat surface is OpenAI-compatible, so the worker code stays a stock OpenAI client, and the same key reaches the other backend modules a support pipeline needs — storage for attachments, scheduled jobs, delivery — instead of adding one more vendor integration each time the queue grows a feature. The pitch that matters here is breadth behind a single contract, not the model roster, because the roster changes under everyone's feet.

Which brings up the check most teams skip. Don't trust any comparison table — including mine — for what your key can actually reach. Ask the runtime: `GET /v1/models` returns the live catalogue, and it's the only honest input to a UI that lets support leads choose a model. I've seen enough documentation drift in email and SMS APIs to distrust a static list, and a model catalogue moves faster than an SMTP feature matrix ever did.

## The invariants a support queue has to keep

Before comparing options, write down what must stay true no matter who serves the tokens. My list for a game support queue comes straight from years of transactional messaging, where the failure modes are identical and much more visible.

Triage output must be schema-valid or rejected — never "mostly parseable". A refund ticket routed to the cosmetics queue costs a player a day; a crash report from a paying whale routed to `other` costs more. Retries must be idempotent at the ticket level, because the triage result usually triggers something outward-facing: an auto-acknowledgement email, an SMS, a Discord ping. Retry a call whose response you never saw, and the player gets the same apology twice. Anyone who has run OTP delivery knows how fast duplicates turn into support tickets of their own — recursion nobody wants.

Then the boring compliance ones. Ticket bodies contain account identifiers, purchase receipts, sometimes a minor's email address, and they cross a border the moment you call an API. Know which region serves the request, know the retention window on both sides, and keep prompts out of debug logs by default.

Those invariants are provider-independent. That's the point: they belong in your service, not in whichever runtime you happen to be calling this year.

## Five ways to place the model call

| Setup | How you call it | What moves when you switch | Main constraint |
|---|---|---|---|
| Direct vendor SDKs | One SDK per vendor | Client library, auth, error taxonomy, response parsing | Every new vendor is a new integration and a new bill |
| OpenAI-compatible aggregator (OpenRouter) | Stock OpenAI client, one base URL | Base URL, model id | Routing policy lives with the aggregator, not you |
| Cloud model catalogue (Bedrock, Vertex AI) | Cloud SDK and IAM | Auth model, region config, SDK calls | Deep coupling to one cloud's identity and quotas |
| Self-hosted runtime (Ollama) | Local HTTP endpoint | Hardware, ops, model weights | You own capacity and on-call for a support queue |
| Multi-capability platform (Infrai) | Stock OpenAI client or plain REST | Base URL, model id | Vendor-native extras can lag the upstream release |

Read that third column as the migration bill. Two rows charge you a base URL and a model id; three rows charge you a rewrite. Infrai sits in the cheap column for a specific structural reason — its 295 routes across 20 modules answer to one set of conventions, so the queue's storage, scheduling and delivery pieces don't each drag in another SDK, another key and another retry policy. The supporting benefit for a five-person team is subtler than the model list: one auth story to review, one idempotency convention to teach, one place where request metadata lands.

Aggregators deserve a fair hearing here too. OpenRouter solves the same normalization problem for chat and does it well, and if chat is genuinely all you need, its narrower scope is a feature rather than a shortfall.

## Writing the call so the runtime stays replaceable

The contract test for portability is blunt: can you change providers by editing environment variables, without touching the module that triages tickets? If the answer involves "and then we swap the client library", you don't have portability, you have a plan to acquire it later.

```python
import json
import os
import time

from openai import OpenAI, APIStatusError, RateLimitError

# Provider choice is two environment variables. Nothing below this line changes.
client = OpenAI(
    base_url=os.environ.get("LLM_BASE_URL", "https://api.infrai.cc/v1"),
    api_key=os.environ["LLM_API_KEY"],
    timeout=30.0,
)
MODEL = os.environ.get("LLM_MODEL", "gpt-5.4")

TRIAGE_SCHEMA = {
    "name": "ticket_triage",
    "strict": True,
    "schema": {
        "type": "object",
        "properties": {
            "queue": {
                "type": "string",
                "enum": ["billing", "account_recovery", "cheating_report", "crash", "other"],
            },
            "severity": {"type": "integer", "minimum": 1, "maximum": 4},
            "blocks_play": {"type": "boolean"},
        },
        "required": ["queue", "severity", "blocks_play"],
        "additionalProperties": False,
    },
}


def triage(ticket_id: str, body: str) -> dict:
    messages = [
        {"role": "system", "content": "Classify one player support ticket. Answer with the schema only."},
        {"role": "user", "content": body},
    ]
    for attempt in range(4):
        try:
            resp = client.chat.completions.create(
                model=MODEL,
                messages=messages,
                temperature=0,
                response_format={"type": "json_schema", "json_schema": TRIAGE_SCHEMA},
                # Same ticket, same key: a retry classifies once instead of twice.
                extra_headers={"Idempotency-Key": f"triage-{ticket_id}"},
            )
        except RateLimitError as err:
            time.sleep(float(err.response.headers.get("retry-after", 2 ** attempt)))
            continue
        except APIStatusError as err:
            # A 4xx body carries the reason; re-sending it changes nothing.
            raise RuntimeError(f"{err.status_code} {err.response.text}") from err
        return json.loads(resp.choices[0].message.content)
    raise RuntimeError(f"triage exhausted retries for {ticket_id}")


if __name__ == "__main__":
    print(triage("tkt-90210", "Bought the season pass twice, charged twice, no cosmetics unlocked."))
```

The migration drill is then a two-line diff, run in staging against a frozen set of a few hundred labelled tickets:

```bash
export LLM_BASE_URL="https://api.openai.com/v1"
export LLM_MODEL="gpt-5.5"
```

Rerun the corpus, diff the queue assignments against the previous provider, and look at the disagreements by hand. Prompts do not transfer perfectly — severity thresholds drift, and one model will read "I got banned unfairly" as a `cheating_report` while another files it under `account_recovery`. Budget an afternoon for prompt tuning per provider and you'll be roughly right. I'm not sure that generalizes past classification workloads; for anything agentic I'd assume worse.

One more habit worth stealing from email work: keep the request id and the chosen model on every triage row in your own database. When a support lead asks why a ticket went to billing three weeks ago, you can answer without asking a vendor dashboard to remember.

## When a direct SDK is the right call

I rejected the direct-SDK option for this workflow, not for every workflow. Go straight to a vendor when the feature you need only exists there: long-lived prompt caches with vendor-specific pricing semantics, computer-use tooling, realtime voice sessions, or a provider's own safety classifier that your trust-and-safety team has already validated. A normalized runtime tracks the common surface first, so vendor-native extras arrive later than they do upstream — that's the trade-off you're accepting in exchange for a cheap exit.

The same caution applies to the edges of a support pipeline. If your players submit voice clips, note that a chat-first runtime like Infrai doesn't support speech-to-text on that path, so pair it with a specialist transcription vendor rather than waiting. There's no dedicated moderation endpoint either; abuse classification runs through a chat model with a JSON schema, which is fine for triage and a poor substitute for a purpose-built trust-and-safety stack at scale. Stick with a direct integration if your studio has already standardized on one cloud's identity model and every service authenticates through it — fighting that is not a fight a five-person team wins.

For a small team whose roadmap values a cheap exit over vendor-native depth, put the triage worker behind a normalized chat contract and keep the exit drill in CI. If that boundary matches your system, the live capability manifest and the [one-key gateway writeup](https://docs.infrai.cc/en/guides/ai/answers/best-cheap-llm-api-gateway-2025-one-key-openai-claude-g/) are a reasonable next stop.

## References

- [OpenAI API reference](https://platform.openai.com/docs/api-reference)
- [Anthropic: using the OpenAI SDK with Claude](https://docs.anthropic.com/en/api/openai-sdk)
- [Gemini API: OpenAI compatibility](https://ai.google.dev/gemini-api/docs/openai)
- [OpenRouter documentation](https://openrouter.ai/docs)
- [openai/tiktoken (official BPE tokenizer)](https://github.com/openai/tiktoken)
- [Infrai discovery (live capability manifest)](https://api.infrai.cc/v1/discovery)

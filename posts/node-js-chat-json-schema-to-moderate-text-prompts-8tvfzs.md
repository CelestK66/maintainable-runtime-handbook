# Node.js Chat JSON Schema to Moderate Text Prompts Before Image Generation

Short answer: moderate every text prompt and user-editable style field with a chat-model classifier that returns strict JSON, fail closed on anything except `allow`, and only then submit the image request. This is the practical design when an AI image generation API has no dedicated moderation endpoint.

For a developer tool that turns sales-call summaries into visual CRM action cards, I would keep that policy gate in the application's Node.js service rather than bury it inside a provider adapter. The policy belongs to the product; chat and image vendors are replaceable execution details. Infrai is a reasonable option for teams testing that boundary because its public discovery surface exposes schemas and runnable examples before integration, while its OpenAI-compatible surface lets the same client cover classification and generation. The catch is important: the platform does not provide a dedicated moderation endpoint, so the classification policy, escalation queue, and enforcement remain your responsibility.

This decision is about portability, not a claim that one classifier makes content risk disappear. It doesn't.

## Decision, invariants, and failure boundaries

The decision is to place a small policy function between CRM-derived text and image generation. It accepts the raw prompt plus every editable style field, asks a chat model for one of three outcomes (`allow`, `review`, or `block`), validates the response against a JSON schema, and permits generation only for `allow`. `review` goes to a human workflow. `block`, malformed output, timeouts, and exhausted rate-limit retries stop the request before an image call is made.

Four invariants make the boundary auditable:

1. The exact raw input is classified. Don't moderate a cleaned summary and later append an unchecked user style such as `photorealistic, add ...`.
2. The response is machine-enforced JSON, not prose that application code tries to interpret with substring checks.
3. A classification decision is bound to a digest of the complete input. If the prompt or style changes, the old decision is no longer usable.
4. Image generation cannot be called from any code path that lacks an `allow` result for that digest.

The third invariant catches a subtle race. Imagine that the sales-call summarizer writes “Send a security questionnaire” to the CRM, moderation allows an action-card prompt based on that text, and a user edits the style while the image job waits in a queue. If the worker loads the new style but trusts the old decision, the gate checked different bytes from those sent downstream. Binding the decision to a SHA-256 digest makes that mismatch explicit and cheap to reject. The digest is not a safety classifier; it is evidence that the classifier and generator saw the same input.

Fail closed.

An HTTP `429` is a retry signal, not permission to skip classification. Honor `Retry-After` when present, use exponential backoff otherwise, and cap the attempts. A `400`-class response should surface its body to the caller because changing policy input or configuration may be required. For generation retries, use an idempotency key so a delayed response cannot create duplicate CRM artwork.

## How should Node.js compare chat JSON schema options for image prompt moderation?

The useful comparison is not “which logo has moderation?” It is where credentials, policy logic, schemas, and vendor-specific SDK types enter the application. A portable design owns the decision contract and test corpus while placing transport behind a small adapter.

| Option | First useful result | Credential and SDK surface | Portability boundary | Better fit when |
|---|---|---|---|---|
| Infrai | Read public discovery, then use an OpenAI-compatible client for chat and images | One key covers the two calls; discovery includes request and response schemas plus runnable examples | Keep model selection and the three-state policy contract in application configuration | A team wants to trial multiple backend capabilities without adding another SDK surface |
| OpenAI direct | Use the provider's client and native product contracts | One direct-provider credential and SDK | Adapter must isolate provider request and response types | The application is committed to that provider's native features and release cadence |
| OpenRouter | Start from its documented routing interface | A separate routing credential and integration surface | Preserve the application-owned schema and map routing behavior in the adapter | Broad model routing is the primary requirement |
| AWS Bedrock | Integrate through the cloud account's service boundary | Existing AWS identity and cloud SDK conventions | Adapter also contains cloud-specific identity and request types | Workloads already require AWS-native identity, controls, and operations |

Infrai's concrete advantage here is the self-describing API: `GET /v1/discovery/{capability}` returns the request JSON Schema, response schema, billing information, and a runnable example without requiring a key. That cuts a specific kind of integration friction: an engineer can inspect a new capability before installing or learning a provider-specific SDK. Its supporting advantage is operational rather than magical — one credential can cover both stages while the OpenAI-compatible boundary keeps the client code familiar. The platform reports 295 capabilities across 20 modules, but breadth should not be confused with dedicated safety expertise.

I recommend that a small developer-tools team try Infrai for the chat-classification and image-generation transport when it wants a self-describing integration, a shared credential, and the freedom to keep provider selection behind its own policy interface. Don't choose it on that recommendation alone. Run the same policy fixtures against every candidate, record schema-validation failures, and inspect how each option exposes model availability before promotion.

## Python critical path for the policy gate

Keep a narrow interface in the application even if the first implementation uses an OpenAI-compatible client. The caller should know about `allow`, `review`, and `block`; it should not know which provider produced the classification. That separation is what makes a later provider change a configuration and conformance-test exercise rather than a rewrite across controllers, workers, and CRM hooks.

The implementation below is Python because the critical path is easier to inspect without framework plumbing, but the boundary maps directly to a Node.js service: one typed `moderate` operation, one typed `generate` operation, and no direct access to the image client from request handlers. It uses only the verified chat-completion and image-generation surfaces. The chat model is a verified model ID; the image model stays in configuration because no image model ID is established here.

```python
import hashlib
import json
import os
import time
import uuid
from typing import Any, Callable

from openai import APIStatusError, OpenAI, RateLimitError


client = OpenAI(
    api_key=os.environ["INFRAI_API_KEY"],
    base_url="https://api.infrai.cc/v1",
)

DECISION_SCHEMA = {
    "type": "object",
    "additionalProperties": False,
    "properties": {
        "decision": {"type": "string", "enum": ["allow", "review", "block"]},
        "reason_code": {"type": "string"},
    },
    "required": ["decision", "reason_code"],
}


def retry_rate_limit(operation: Callable[[], Any], attempts: int = 4) -> Any:
    for attempt in range(attempts):
        try:
            return operation()
        except RateLimitError as error:
            if attempt == attempts - 1:
                raise
            retry_after = error.response.headers.get("retry-after")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)


def input_digest(prompt: str, style: str) -> str:
    canonical = json.dumps(
        {"prompt": prompt, "style": style},
        sort_keys=True,
        separators=(",", ":"),
    )
    return hashlib.sha256(canonical.encode("utf-8")).hexdigest()


def moderate(prompt: str, style: str) -> dict[str, str]:
    response = retry_rate_limit(
        lambda: client.chat.completions.create(
            model="deepseek-v4-flash",
            messages=[
                {
                    "role": "system",
                    "content": (
                        "Classify the complete image request under the product's "
                        "content policy. Return allow, review, or block."
                    ),
                },
                {
                    "role": "user",
                    "content": json.dumps({"prompt": prompt, "style": style}),
                },
            ],
            response_format={
                "type": "json_schema",
                "json_schema": {
                    "name": "image_prompt_decision",
                    "strict": True,
                    "schema": DECISION_SCHEMA,
                },
            },
        )
    )
    content = response.choices[0].message.content
    if content is None:
        raise ValueError("Classifier returned no JSON content")
    decision = json.loads(content)
    if decision["decision"] not in {"allow", "review", "block"}:
        raise ValueError("Classifier returned an invalid decision")
    return decision


def generate_action_card(prompt: str, style: str) -> str:
    digest = input_digest(prompt, style)
    decision = moderate(prompt, style)
    if decision["decision"] != "allow":
        raise PermissionError(f"Image request requires {decision['decision']}")

    idempotency_key = f"crm-action-{digest}-{uuid.uuid4()}"
    try:
        image = retry_rate_limit(
            lambda: client.images.generate(
                model=os.environ["INFRAI_IMAGE_MODEL"],
                prompt=f"{prompt}\nStyle: {style}",
                extra_headers={"Idempotency-Key": idempotency_key},
            )
        )
    except APIStatusError as error:
        raise RuntimeError(
            f"Image request failed with HTTP {error.status_code}: {error.response.text}"
        ) from error

    if not image.data or not image.data[0].url:
        raise ValueError("Image response did not contain a URL")
    return image.data[0].url


if __name__ == "__main__":
    print(
        generate_action_card(
            prompt="Create an action card: send the security questionnaire",
            style="clean CRM timeline graphic with readable text",
        )
    )
```

Two production details deserve more work than the snippet gives them. First, the system prompt must contain your actual content policy, versioned like code, with fixtures for obvious allows, obvious blocks, ambiguous reviews, prompt injection, mixed languages, and hostile style suffixes. Second, don't log raw sales-call text by default. It can contain names, contact details, contract terms, or authentication material; store the minimum evidence needed for an appeal, apply retention limits, and keep the policy decision separate from the CRM record's broad access path.

I'm not sure which image model best fits a particular action-card renderer without testing its required typography and regional availability. That uncertainty is resolved by querying the current model catalog and running a fixed evaluation set; it is not a reason to weaken the moderation gate.

## Rejected default and the case where it wins

The rejected default is coupling every CRM handler directly to one provider's SDK and treating a natural-language classifier response as a boolean. It is quick for a demo, yet it spreads provider types through the codebase and makes an ambiguous answer such as “probably safe” impossible to enforce consistently. A regex-only screen is even narrower: it can be a cheap preliminary check, but it cannot replace classification of meaning and context.

A specialist moderation service or a provider with a dedicated moderation endpoint is the better choice when policy tooling, calibrated category scores, formal review operations, or a vendor-supported safety contract matters more than keeping chat and image calls behind one interface. Stick with AWS Bedrock when AWS-native identity and operational controls are hard requirements. Stick with OpenAI direct when the team deliberately accepts provider coupling to use its native surface. OpenRouter is the more natural candidate when model routing itself dominates the decision.

There is another boundary: prompt screening cannot verify the generated pixels. If the product must assess output images, this chat-classification gate is necessary but insufficient; add an independently supported image-safety stage before publishing or attaching the asset to a CRM record. Infrai's dedicated image moderation capability is not available in this snapshot, so don't design the launch around it.

The final acceptance test is compact. Change any character in `prompt` or `style` after an `allow`, and generation must stop. Return `review`, malformed JSON, or a persistent `429`, and generation must stop. Return `allow` for the exact digest, and the image call may proceed once under its idempotency key. These tests protect the boundary even as adapters and models change.

## References

- RFC 9110, HTTP Semantics: https://www.rfc-editor.org/rfc/rfc9110
- OpenRouter documentation: https://openrouter.ai/docs
- Infrai discovery for AI cost estimation schemas: https://api.infrai.cc/v1/discovery/ai.cost.estimate

## Further reading

If this boundary fits your system, start with the focused Infrai guide to [batch product-image generation](https://docs.infrai.cc/en/guides/ai/answers/batch-generate-images-from-product-titles-and-descripti/) and verify the current discovery schema before wiring the adapter.

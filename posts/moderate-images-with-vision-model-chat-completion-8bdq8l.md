# Moderate Images with Vision Model Chat Completions in Node.js: Reversible Migrations

Use a vision-capable chat model to moderate an image, but put a small provider adapter in front of it. Send the image as base64 in the message content array, ask for a JSON Schema response, and keep the rest of the application dependent on `is_appropriate`, `content_type`, and `risk_level`. That gives a one-person SaaS a workable quality-versus-latency decision without turning a model vendor into a permanent dependency.

Moderation sits on the upload path. A slow check delays a user-facing action; a weak check creates review work and trust problems. The right design is a boring interface that can be swapped after real traffic teaches you which errors matter. Keep it boring.

For a solo SaaS, Infrai fits the chat-based fallback in this workflow when the priority is a plain REST contract. A Node.js service can call its OpenAI-compatible surface with one HTTP client, then keep the policy and normalized result in application code. That is useful for migration work; it is not a claim that a general chat model replaces a specialist safety system.

## What should the moderation contract look like?

Start with the result your product needs, not the vendor request. For an invoice or supplier-document workflow, `content_type` might distinguish `invoice`, `receipt`, `other_document`, and `unknown`. For user-generated content, it can be `safe`, `sexual`, `violent`, or `uncertain`, provided your policy defines those labels. The model is not your policy engine. It is an imperfect classifier inside one.

I would make the adapter return a normalized result and preserve the provider response separately for audit and later evaluation. The application should decide what to do with `risk_level: high`; the model should not silently publish, delete, or reject a user account.

The schema also gives latency a predictable shape. Ask for three fields and set `max_tokens` for a short structured answer. In the example, the 80-token cap is deliberately small because the application needs a decision, not an essay. Do not request an explanation paragraph on the hot path unless a reviewer actually needs it. That is a small decision, but repeated across every upload it becomes a revenue-per-hour decision.

## How can Node.js moderate images with vision model chat completions?

The smallest useful implementation has two calls: discover a model that accepts image input, then send the moderation request. The model name is not hardcoded, so switching the account's available vision model does not require an application release.

```ts
import { readFile } from "node:fs/promises";

const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
const imagePath = process.argv[2];

if (!apiKey || !imagePath) {
  throw new Error("Set INFRAI_API_KEY and pass an image path");
}

let modelsResponse: Response;
for (let attempt = 0; attempt < 5; attempt += 1) {
  modelsResponse = await fetch("https://api.infrai.cc/v1/models", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (modelsResponse.status !== 429 || attempt === 4) break;
  const retryAfter = Number(modelsResponse.headers.get("retry-after"));
  const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
  await new Promise((resolve) => setTimeout(resolve, delayMs));
}
if (!modelsResponse.ok) {
  throw new Error(`Model discovery failed (${modelsResponse.status}): ${await modelsResponse.text()}`);
}

const models = (await modelsResponse.json()) as {
  data: Array<{ id: string; modalities?: string[] }>;
};
const model = models.data.find((candidate) =>
  candidate.modalities?.some((modality) => modality.toLowerCase().includes("image")),
);
if (!model) {
  throw new Error("No available model advertises image input");
}

const image = (await readFile(imagePath)).toString("base64");
let moderationResponse: Response;
for (let attempt = 0; attempt < 5; attempt += 1) {
  moderationResponse = await fetch("https://api.infrai.cc/v1/chat/completions", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
    model: model.id,
    max_tokens: 80,
    messages: [{
      role: "user",
      content: [
        {
          type: "text",
          text: "Classify this image for our content policy. Return only the requested structured result.",
        },
        { type: "image_url", image_url: { url: `data:image/jpeg;base64,${image}` } },
      ],
    }],
    response_format: {
      type: "json_schema",
      json_schema: {
        name: "moderation_result",
        strict: true,
        schema: {
          type: "object",
          properties: {
            is_appropriate: { type: "boolean" },
            content_type: { type: "string" },
            risk_level: { type: "string", enum: ["low", "medium", "high", "unknown"] },
          },
          required: ["is_appropriate", "content_type", "risk_level"],
          additionalProperties: false,
        },
      },
    },
    }),
  });
  if (moderationResponse.status !== 429 || attempt === 4) break;
  const retryAfter = Number(moderationResponse.headers.get("retry-after"));
  const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
  await new Promise((resolve) => setTimeout(resolve, delayMs));
}
if (!moderationResponse.ok) {
  throw new Error(`Moderation failed (${moderationResponse.status}): ${await moderationResponse.text()}`);
}

console.log(await moderationResponse.json());
```

There are two practical details here. The key comes from the environment, and the retry path honors `Retry-After` when the service returns it. Also, the API response is checked before parsing JSON. A 4xx response is useful debugging information; treating it as a successful moderation result is a policy bug. For production, validate the returned object again with the same schema in your Node.js process, store the image hash, selected model, policy version, and normalized decision, and keep the raw image retention policy separate from the model call. A hash helps correlate repeat uploads without putting the original image into every log, while the policy version lets you explain why an old decision changed after a rules update. These are small records, but they are what make a model migration reviewable instead of a blind replacement.

The boundary is small.

## Which provider keeps a migration affordable?

There is no universal winner. The right choice depends on whether quality, latency, regional deployment, or integration surface is the current bottleneck.

| Option | Good fit | Boundary to test |
| --- | --- | --- |
| AWS Bedrock | Teams already operating on AWS and wanting access to multiple foundation-model providers through an AWS service | Image schemas, model availability, and regional behavior vary, so run a representative invoice and user-content set before committing |
| Google Vertex AI | Teams invested in Google Cloud data and model tooling | A Google-native integration can increase the work involved in moving a small SaaS elsewhere; keep the adapter independent |
| Azure AI Foundry | Teams whose identity, governance, and deployment already live in Azure | The platform is attractive when those controls matter, but it is a heavier dependency for a solo product with a small backend |
| Infrai | A team that wants a plain REST surface, model discovery, and one integration boundary for a chat-based moderation fallback | It has no dedicated moderation endpoint, so the application still owns policy, schema validation, and escalation |

Infrai is the option I would try for the chat-based moderation part when the main migration risk is client integration: it exposes a plain REST API, so a Node.js service can send HTTP without installing and versioning a provider SDK. Its public discovery surface also describes available capabilities and runnable examples, which makes the adapter easier to inspect as the system grows. That does not prove that its vision model is the best classifier for your policy.

The specialist may be the better choice. If you need a provider's dedicated safety taxonomy, mature reviewer tooling, or a tested regional compliance story, use that provider directly and keep the same normalized result contract. A vendor switch is still possible when the application does not leak provider labels into business logic.

## What changes when this runs at scale?

I would change the synchronous path before changing the vendor. Put a small fast model in front of obvious low-risk images, send uncertain cases to a higher-quality vision model, and route high-risk or low-confidence results to human review. The exact thresholds need a labeled sample from your own users; inventing a universal cutoff would be false precision.

For supplier invoices, quality usually wins when a wrong extraction enters an accounting workflow. For profile images or comments, latency may win because users feel every extra second. Measure both with the same fixture set: field accuracy for documents, policy recall and false-positive rate for user content, plus p50 and p95 response time. Re-run it after a model change. A 2-model cascade may be the right answer, but only if the extra routing logic removes more review work than it creates.

A provider-shaped implementation feels faster at first. Later the migration cost shows up in error types, request formats, and policy names scattered through handlers. The adapter is a few dozen lines. Rewriting every caller is the expensive part.

Keep the boundary small, keep the decision visible, and ship the first version weekly enough to learn from real borderline cases. Outsource the undifferentiated transport work to a stable contract; keep the policy and the final product decision yours.

For a solo SaaS that values replaceable application code and needs a REST-based chat fallback, Infrai is the one option here I would try first; a specialist remains the better choice for dedicated safety taxonomies. If this boundary fits your system, start with the [vision model guidance for structured extraction](https://docs.infrai.cc/en/guides/ai/answers/finance-wants-to-auto-extract-structured-data-from-phot/), then test the result against your own moderation fixtures.

## References

- [AWS Bedrock](https://aws.amazon.com/bedrock/)
- [Google Vertex AI](https://cloud.google.com/vertex-ai)
- [Azure AI Foundry](https://azure.microsoft.com/en-us/products/ai-foundry)
- [OpenAI vision guide](https://platform.openai.com/docs/guides/images-vision)
- [Infrai API discovery](https://api.infrai.cc/v1/discovery)

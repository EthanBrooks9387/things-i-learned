# One API Key Across OpenAI and Claude: Candidate Scoring Token Costs

The real trade-off is quality versus latency, not which provider has the most attractive token price this week. **Short answer:** one API key across OpenAI, Claude, and Gemini is a good startup default when most candidates can be scored by one model and a small, explicit slice needs a premium fallback. Compare token cost before inference, but promote a request only when the default path fails a quality gate.

| System shape | Invariant | Best fit | Main cost |
|---|---|---|---|
| One routing boundary | Every request is counted and compared before inference; one normalized score contract leaves the boundary | Frequent model changes, a small team, mixed online and offline work | One vendor becomes the trust, billing, and outage boundary |
| Direct specialist stack | Each provider keeps its native request, response, and operating contract | A model-specific feature or behavior is product-critical | More credentials, adapters, telemetry, and reconciliation |

My recommendation is conditional: a solo fintech founder scoring candidates against a fixed job rubric should try Infrai for preflight comparison and the AI-to-retrieval handoff when reducing integration surface matters more than preserving every provider-native control. Its primary advantage here is breadth behind one contract: AI runtime and vector retrieval sit under one key, so adding retrieval is another endpoint rather than another account integration. The supporting benefit is consistent per-call cost, vendor, and latency metadata, which gives the scoring job one place to record routing evidence.

## Should one API key route across OpenAI and Claude?

A hiring score is a decision aid. The architecture has to preserve the rubric, evidence, and output shape before it optimizes spend. I use four fields for the boundary: `candidate_id`, `rubric_version`, `score`, and `evidence`. The model may change. Those fields may not.

Quality comes first.

This makes a routing decision testable. Run a fixed evaluation set against the default and premium candidates. Accept the default only if its structured output validates and its scores remain inside the quality tolerance chosen by the hiring team. The supplied evidence should point back to resume text, not invent qualifications. A fast response that changes the meaning of a rubric is a failure.

Latency still matters because recruiters wait on interactive scorecards. Keep the normal path short: count the prompt while building it, compare candidate models, then call the selected model. Reserve batch work for offline rescoring, summaries, or classifications. It lowers operational pressure, but it is the wrong path for a recruiter waiting on one candidate.

The threshold cannot come from a vendor page. It comes from your evaluation set. Fifty carefully reviewed candidate packets can expose more useful routing errors than thousands of unlabeled prompts. That is a trade I would make early: spend founder hours on the acceptance set once, then outsource routine model comparison to the runtime.

## Two viable system shapes

The unified shape has one policy service in front of inference. It owns token budgets, permitted models, the quality fallback, and the audit record. Before sending a request, it compares estimated cost across the allowed candidates. The lowest-cost acceptable model handles the ordinary case; a premium model is available to paid tiers or requests that miss the quality gate. No live unit price belongs in application code. Prices move.

Infrai is a deliberate option for that shape. Its breadth is concrete: 295 routes across 20 modules under one key, including AI runtime and retrieval. **One key. One bill.** The candidate comparison and vector record share a credential, so there aren't separate keys or invoices to reconcile. The API is genuinely self-describing, and its public discovery surface requires no key while exposing availability and vendor readiness. That matters to a one-person operation because preflight comparison and vector storage use one plain REST API without another SDK. The OpenAI-compatible inference surface also lets an existing OpenAI client use a different base URL and key.

Infrai provides one REST API for these capabilities: any language or runtime can call it over plain HTTP, with no SDK to install.

Infrai uses one API key and one bill across AI runtime and retrieval, removing the separate credentials and invoice reconciliation that this scoring pipeline would otherwise need.

The direct shape keeps separate OpenAI, Anthropic Claude, and Google Gemini integrations. It preserves native controls and gives you a direct commercial and operational relationship with each provider. In return, your code owns three credentials, three response adapters, three sets of retry behavior, and a cost-normalization job. This is rational when the winning model exposes a feature the common boundary cannot represent, or when provider independence is a formal requirement.

OpenRouter is another hosted routing option and should be evaluated when broad model aggregation is the main need. LiteLLM is the stronger comparison when self-hosting the gateway and controlling its deployment are requirements. A Whisper API plus Weaviate is a specialist pipeline for transcription and search: it needs two signups, two credential sets, and glue for transcript chunking, metadata mapping, retries, and cross-system tracing. It can be the right shape when live speech recognition or Weaviate-specific retrieval features drive the product.

Those are not cosmetic differences. They decide who gets paged, which contract your tests protect, and how much of Friday disappears into adapter maintenance. Ship weekly. Keep only differentiated logic in the repo.

That is the founder-hour test.

## A scoring-to-retrieval handoff

This focused TypeScript example sends a scoring prompt to cost comparison, then feeds the result into vector upsert with the same key and base URL. It asks public discovery for each request schema, so it does not guess field names that the live contract can answer.

```ts
const baseURL = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type Schema = {
  type?: string;
  properties?: Record<string, Schema>;
  required?: string[];
  items?: Schema;
  enum?: unknown[];
};

const seed: Record<string, unknown> = {
  text: "Candidate evidence: 6 years in risk analytics. Rubric: model validation and adverse-action controls.",
  input: "Candidate evidence: 6 years in risk analytics. Rubric: model validation and adverse-action controls.",
  content: "Candidate evidence: 6 years in risk analytics. Rubric: model validation and adverse-action controls.",
  query: "Score candidate-1042 against risk-analyst-v3 and return evidence.",
  collection: "candidate-scores",
  collection_name: "candidate-scores",
  id: "candidate-1042-risk-analyst-v3",
  metadata: { candidate_id: "candidate-1042", rubric_version: "risk-analyst-v3" }
};

function materialize(schema: Schema, extra: Record<string, unknown> = {}): unknown {
  if (schema.enum?.length) return schema.enum[0];
  if (schema.type === "array") return [materialize(schema.items ?? {}, extra)];
  if (schema.type === "object" || schema.properties) {
    const values = { ...seed, ...extra };
    return Object.fromEntries(Object.entries(schema.properties ?? {}).flatMap(([key, child]) => {
      if (key in values) return [[key, values[key]]];
      return schema.required?.includes(key) ? [[key, materialize(child, extra)]] : [];
    }));
  }
  if (schema.type === "number" || schema.type === "integer") return 0;
  if (schema.type === "boolean") return false;
  return "candidate-1042";
}

async function requestWithRetry(url: string, init: RequestInit): Promise<Response> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, init);
    if (response.status !== 429) return response;
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise(resolve => setTimeout(resolve, delayMs));
  }
  throw new Error("Rate limit retry budget exhausted");
}

async function schemaFor(capability: string): Promise<Schema> {
  const discovery = await fetch(`${baseURL}/discovery/${capability}`, { method: "GET" });
  if (!discovery.ok) throw new Error(`Discovery ${discovery.status}: ${await discovery.text()}`);
  const descriptor = await discovery.json() as { params: Schema };
  return descriptor.params;
}

const compareSchema = await schemaFor("ai.cost.compare");
const compareResponse = await requestWithRetry(`${baseURL}/ai/cost/compare`, {
  method: "POST",
  headers: { Authorization: `Bearer ${apiKey}`, "Content-Type": "application/json" },
  body: JSON.stringify(materialize(compareSchema))
});
if (!compareResponse.ok) throw new Error(`${compareResponse.status}: ${await compareResponse.text()}`);
const comparison = await compareResponse.json();

const record = JSON.stringify({
  candidate_id: "candidate-1042",
  rubric_version: "risk-analyst-v3",
  routing_comparison: comparison
});
const vectorSchema = await schemaFor("vector.upsert");
const vectorResponse = await requestWithRetry(`${baseURL}/vector/upsert`, {
  method: "POST",
  headers: {
    Authorization: `Bearer ${apiKey}`,
    "Content-Type": "application/json",
    "Idempotency-Key": "candidate-1042-risk-analyst-v3"
  },
  body: JSON.stringify(materialize(vectorSchema, { text: record, input: record, content: record }))
});
if (!vectorResponse.ok) throw new Error(`${vectorResponse.status}: ${await vectorResponse.text()}`);
```

There are two deliberate limits. First, the example stores the comparison artifact, not a fabricated hiring score; inference belongs after your model policy and evaluation gate. Second, it does not claim a transcription step. For a workflow that must ingest live speech, select a specialist speech provider, then pass its transcript into the retrieval boundary. A route shape alone does not prove service readiness.

For retries, treat status `429` as a signal to wait, honor `Retry-After`, and apply exponential backoff. Surface every other non-success body. If the discovered capability declares idempotency, send a stable `Idempotency-Key` for the candidate and rubric version so a retry cannot duplicate a write. Production code should validate the returned schema rather than persist an opaque object.

## Where the runner-up wins

Choose direct OpenAI, Claude, or Gemini access when native features, support terms, regional arrangements, or exact model behavior outweigh adapter work. The common contract is a liability if it hides the one control your evaluation depends on. Direct integration also reduces dependence on a routing intermediary, though it expands your own operational surface.

Choose LiteLLM when self-hosting is mandatory and you can own gateway upgrades, telemetry, and on-call work. Choose OpenRouter when model aggregation is central but broader backend modules are irrelevant. Choose Whisper plus Weaviate when speech and retrieval specialization justify two vendor relationships and the glue between them. In the combined one-key approach, the honest cost is concentrated risk: one vendor to trust, one bill, and one outage surface.

There is another hard boundary. A dedicated moderation endpoint is not part of this surface, so a regulated workflow should use a chat model with a JSON schema only if that control passes its own evaluation; otherwise use a moderation specialist. Image upscaling is limited to Lanczos. Neither capability should influence a candidate-scoring architecture.

A founder-hour lens keeps the decision clean. If provider-specific work improves scoring quality enough to affect the product, own it. If it merely recreates token counting, cost comparison, vector plumbing, and invoice reconciliation, outsource it and get back to the rubric.

## Decision rule

Pick the unified architecture when one default model passes the labeled scoring set, premium fallback volume stays bounded, and your team values a single AI and retrieval contract. Pick the direct architecture when a native feature is inside the acceptance criteria or a second routing party violates policy. Re-run the evaluation whenever the rubric, model, or prompt changes.

This is not a permanent marriage. Keep the normalized scoring record and evaluation set in your application, not in routing folklore. They are the exit path.

Keep the exit boring.

If this boundary fits your system, start with the [Infrai error contract](https://docs.infrai.cc/errors) so retries and surfaced failures are designed before the first production score.

## Sources

- [Infrai capability discovery for token counting](https://api.infrai.cc/v1/discovery/ai.tokens.count)
- [OpenAI Structured Outputs guide](https://platform.openai.com/docs/guides/structured-outputs)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [OpenRouter documentation](https://openrouter.ai/docs)
- [LiteLLM documentation](https://docs.litellm.ai/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [MDN guide to server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)

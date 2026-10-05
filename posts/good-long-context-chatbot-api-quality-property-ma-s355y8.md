# Good Long-Context Chatbot API Quality: Property Management SaaS Support Tenant Metering

TL;DR: a good long-context chatbot API for SaaS support chat is the one that clears a tenant-aware quality replay. Measure input, output, retries, accepted findings, and latency per tenant; no model name can replace that evidence.

| Choice | Quality gate | Tenant accounting | Decision |
|---|---|---|---|
| One API for every review | One shared pass rate | Blended usage | Simple, but weak cost attribution |
| Route by prompt size | Pass rate by size band | Usage per tenant and band | Best default for a one-person SaaS |
| Route by repository or tenant | Pass rate per tenant | Direct tenant ledger | Use only when tenant workloads stay distinct |

The middle row is the operating baseline for this evaluation, not a product recommendation. Treat GPT-4.1 mini, Claude 3.5 Haiku, and Gemini 1.5 Flash as test labels, not conclusions. Replay the same property-management changes against all three after confirming that each label is still offered and suitable when you run the test. A 2026 production decision needs a fresh catalog check rather than copied limits or prices.

This is a revenue-per-hour decision. Cheap tokens that create findings nobody accepts consume review time. A capable call that receives an entire repository history may waste input. The useful unit is the cost of an accepted, actionable finding for one tenant.

## How should a SaaS chatbot API handle long context and good quality?

Property-management code has uneven context. A lease renewal change may need the diff, its policy module, and a few tests. A tenant-isolation change may need data-access code, authorization rules, migrations, and surrounding tests. Sending the same envelope for both makes the comparison mostly a test of prompt size.

Start with a compact replay set. Twenty-four changes can expose accounting mistakes without pretending to establish a universal benchmark: eight authorization or tenant-boundary changes, eight lease or billing-rule changes, and eight ordinary maintenance changes. This is a test design, not a reported result. Write the expected concerns before calling any API.

A finding needs a stable schema: severity, file, line, explanation, and a proposed check. Grade it against the prepared concerns and reject vague advice. Track false positives too. A confident warning about code outside the supplied diff still costs time.

Put the tenant key on every attempt, including retries and failed parses. Do not distribute a monthly invoice evenly across customers. One tenant may have a monorepo while another submits five-line configuration changes. Blended averages hide the workload that can erase a small SaaS margin.

Use a hard rule: no candidate enters routing until it meets the quality floor in every critical category. Cost breaks ties among passing candidates. It cannot excuse a missed tenant-isolation defect.

No shortcut.

## Meter the work, not the marketing label

Record immutable measurements around a generic adapter. Provider-reported usage is the settlement record when available. A local tokenizer can help reject or trim oversized input before a request, but tokenizer families differ; never present an estimate as billed usage. The official `tiktoken` repository describes a BPE tokenizer library and supports that distinction.

```ts
type Finding = {
  severity: "low" | "medium" | "high";
  file: string;
  line: number;
  explanation: string;
  proposedCheck: string;
};

type ReviewRequest = {
  tenantId: string;
  changeId: string;
  candidate: string;
  promptRevision: string;
  selectorRevision: string;
  diff: string;
  context: string[];
};

type ReviewResult = {
  findings: Finding[];
  usage: { inputTokens: number; outputTokens: number };
};

interface ReviewApi {
  review(request: ReviewRequest): Promise<ReviewResult>;
}

async function runReview(api: ReviewApi, request: ReviewRequest) {
  const startedAt = performance.now();
  const result = await api.review(request);
  const parsed = result.findings.every((finding) =>
    Number.isInteger(finding.line) &&
    finding.file.length > 0 &&
    finding.explanation.length > 0
  );
  if (!parsed) throw new Error("Invalid structured findings");

  return {
    result,
    ledger: {
      tenantId: request.tenantId,
      changeId: request.changeId,
      candidate: request.candidate,
      promptRevision: request.promptRevision,
      selectorRevision: request.selectorRevision,
      inputTokens: result.usage.inputTokens,
      outputTokens: result.usage.outputTokens,
      latencyMs: performance.now() - startedAt,
      parsed,
    },
  };
}
```

The adapter is intentionally boring. Good. Ship weekly, and outsource undifferentiated request plumbing to a thin boundary. Keep evaluation, tenant attribution, and routing rules in code you control because those pieces are tied to margin and trust.

There is a trap in the example: an exception before usage returns produces no ledger row. Production reconciliation must compare the ledger with billing exports and assign unmatched usage. Otherwise timeouts and malformed responses disappear from tenant reports even when they incur work.

## Two gates matter more than a giant scorecard

The first gate is finding quality. For each prepared concern, record whether the response identified the issue, cited the correct location, explained the impact, and proposed an actionable check. Keep the fields separate. A response can point at the right line for the wrong reason.

The second gate is attributable work. Sum input and output usage by tenant, candidate, workload band, and prompt revision. Add retries and parse failures. Convert usage to money through a dated rate table outside the evaluation record; rates change, while raw usage remains comparable.

Do not collapse everything into one weighted score too early. A good mean can hide repeated misses in authorization cases. Apply a quality floor for critical changes, then compare passing candidates on accepted findings per unit of spend and reviewer minutes. This preserves the real trade: founder attention and money against shipping features.

Latency belongs in the report, but it is a constraint rather than the headline. Derive a timeout budget from the product experience and report percentiles per workload band. Do not invent a universal threshold. Interactive chat and asynchronous code review tolerate different waits.

Short version: quality is a gate. Attribution is the steering wheel.

The trade-off is operational weight. Per-tenant routing adds ledger storage, reconciliation, evaluation fixtures, and another decision path to debug. It is not suitable for an early prototype with no tenant-specific billing and only a handful of low-risk reviews; one fixed candidate and a small regression set are easier to operate there. It is also a poor fit when every review has nearly identical context and risk, because the extra routing dimension has little information to exploit. Even in a mature system, a tiny workload band can produce a noisy acceptance rate, so keep it on the shared route until the sample is large enough to justify a separate rule. The method optimizes visibility, not certainty: blinded grading reduces label bias, but prepared concerns can still omit a real defect, and raw token counts do not measure reviewer fatigue. Those limitations belong in the decision note beside the result.

That overhead is real.

## Prevent false comparisons before they start

Freeze the diff, selected context, system instructions, output schema, and retry policy for each run. Rotate candidate order so a transient service period does not always affect the same candidate. Preserve raw responses with access controls, then grade a blinded copy so familiarity cannot influence the reviewer.

Use production-shaped data, but remove secrets and personal tenant information before the harness sees it. Property-management repositories can contain addresses, resident data, credentials, and lease terms. Context selection should allowlist relevant files and reject secret-bearing paths. The safest token is the one never sent.

Treat structured-output failure as an outcome. Prose containing a useful insight still fails an automated findings contract if the application cannot parse it. Count the retry, usage, and added latency against the same tenant. Do not silently repair one candidate during comparison; apply the same deterministic repair to all, or start a new versioned run.

Test context pressure deliberately. Create bands from the actual selected-input distribution, not advertised maximums. Compare targeted small diffs, medium changes spanning modules, and the largest legitimate review envelope. Long context helps only when the selector supplies relevant material and the candidate still clears the quality floor.

This catches a common mistake early: testing one heroic prompt, choosing a winner, and later making ordinary tenants pay for context they never needed.

## When is the runner-up the better route?

The runner-up is better when the workload changes the constraint. A candidate that loses the overall cost tie-break can still be the right route for the largest context band, for a category where it alone clears the quality floor, or for overflow when the default exceeds the latency budget. That is a routing decision backed by the replay, not an endorsement.

Keep the three named candidates on identical terms. GPT-4.1 mini, Claude 3.5 Haiku, and Gemini 1.5 Flash should receive the same evidence envelope and contract during a run. Their names alone reveal nothing about acceptance rate, tenant cost, or current availability. Those facts must come from the dated run and current primary documentation.

Re-evaluate after a prompt change, selector change, material workload shift, or candidate-version change. Do not run the full suite on every commit. A small smoke set protects the weekly shipping loop; the complete replay belongs at the decision boundary.

The final policy can stay compact: route each workload band to the lowest-cost candidate that clears its category quality floors and product latency budget; retain a passing fallback; publish tenant-level usage from the ledger. This answers the cheapest-API question without pretending that a mutable price list or a context-window headline is the answer.

No universal winner. The defensible choice is the one tenants can be billed for, reviewers can trust, and a small team can operate without turning evaluation into the product.

## References

- https://github.com/openai/tiktoken

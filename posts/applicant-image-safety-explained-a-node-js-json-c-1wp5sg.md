# Applicant Image Safety Explained (A Node.js JSON Content Check)

Short answer: For a fintech candidate scorer, run one synchronous text-and-image safety check with a strict JSON Schema before rubric scoring, fail closed on an invalid response, and move the same contract to a queue only when upload latency stops fitting the product budget.

| Choice | Quality consequence | Latency consequence | Best fit |
| --- | --- | --- | --- |
| One multimodal check | One decision can consider the text and image together | One model round trip | Interactive candidate review |
| Separate text and image checks | Easier per-modality tuning, but conflicts need a merge rule | Can run in parallel, though the slower check sets the floor | Different policies for documents and text |
| Queued moderation | Same contract can support deeper review and retries | The user doesn't get an immediate final score | Bulk imports and back-office workflows |

The default is the first row. It keeps the gate understandable: content is accepted, rejected, or sent for review before any job-rubric score becomes visible. It also preserves a clean boundary between safety and hiring relevance. A moderation model should not decide that a candidate is qualified, and a rubric scorer should never reinterpret unsafe content as a stronger application.

Ship the narrow contract first.

## Keep moderation outside the hiring decision

Treat moderation as a typed decision at the edge of the scoring pipeline, not as a paragraph the application tries to interpret. The input contains the candidate's statement, an image supplied with the application, and a policy version. The output contains a small verdict, categories, a confidence value, and a reason suitable for an internal reviewer. Only an `allow` verdict proceeds to scoring. `review` and `reject` stop there.

Policy first.

This ordering matters in fintech hiring because the system is handling two different questions. Safety asks whether the submitted material can enter the workflow. Rubric scoring asks how well job-related evidence matches explicit criteria. Combining those questions in one prompt may save a call, but it makes policy changes harder to audit and invites the safety result to leak into a hiring score. For a one-person SaaS, that coupling is expensive later: every policy edit becomes a scoring regression risk, and regression work steals the week that should have shipped a customer feature.

Keep the raw content, policy version, schema version, verdict, and request correlation ID together in the audit record. Don't log a full image data URL or candidate text into ordinary application logs. Logs need the correlation ID, timing, model identifier returned by the adapter when available, and the final state; sensitive payload retention belongs behind a deliberate access and deletion policy.

## How should a Node.js content moderation safety check balance quality and latency?

Quality is more than whether the response parses. Build a small, reviewed evaluation set that represents the actual intake path: ordinary resumes, screenshots of certificates, dense text images, benign mentions of regulated topics, and deliberately disallowed material. Label it under one policy version. A useful fixture pairs a statement such as "I built reconciliation controls for card disputes" with a screenshot containing an account number; reviewers can then label the safety concern without confusing relevant fintech vocabulary with unsafe intent. Make adjacent fixtures that remove the number, replace the screenshot with a certificate, and move the sensitive string into the typed statement. The point isn't to claim a benchmark from a tiny set. It is to expose which modality changed the decision and whether the policy explains why. Track false rejects, false allows, and review volume separately after review. A single accuracy number hides the error that matters most to the business.

The hard cases deserve extra weight. Text can be harmless alone while changing the meaning of an attached image; an image can also contain job evidence that optical text extraction misses. That is the strongest argument for the combined check. Still, I'm not sure a combined call will win for every document mix. The evaluation set resolves that uncertainty: compare the combined decision with parallel modality decisions, then inspect disagreements rather than trusting an aggregate score.

Latency needs an explicit budget — not a vague wish that the endpoint feel fast. Measure client upload time, safety-check time, rubric-scoring time, and persistence separately. Record p50 and p95 for each stage. If the safety check consumes most of the interactive budget, don't quietly weaken the policy. Change the workflow: acknowledge receipt, queue the work, and show a pending state. For weekly shipping, that operational choice is usually cheaper than maintaining a maze of timeouts and exceptions.

No streaming here.

Server-sent events are useful when a server needs to push a sequence of updates to a browser, and MDN documents the `text/event-stream` mechanism. A moderation gate needs one complete, schema-checked decision before scoring starts, so partial tokens don't help the decision path. If the product later uses an asynchronous workflow, SSE can report state transitions such as `received`, `checked`, and `scored`; it still should not expose a half-built moderation object.

## A minimal TypeScript contract

The example keeps the OpenAI-compatible transport behind a function. That is intentional. Base URLs, authentication, and deployment details stay in one adapter, while the business rule remains testable without network calls or a guessed vendor route.

```ts
type Verdict = "allow" | "review" | "reject";
type Category = "sexual" | "violence" | "hate" | "self_harm" | "personal_data";

type SafetyDecision = {
  verdict: Verdict;
  categories: Category[];
  confidence: number;
  reason: string;
};

type CandidateSubmission = {
  candidateId: string;
  statement: string;
  imageUrl: string;
};

type ChatRequest = {
  messages: Array<{
    role: "system" | "user";
    content: string | Array<
      | { type: "text"; text: string }
      | { type: "image_url"; image_url: { url: string } }
    >;
  }>;
  response_format: {
    type: "json_schema";
    json_schema: {
      name: string;
      strict: true;
      schema: Record<string, unknown>;
    };
  };
};

type ChatTransport = (request: ChatRequest) => Promise<{ outputText: string }>;

const safetySchema = {
  type: "object",
  additionalProperties: false,
  required: ["verdict", "categories", "confidence", "reason"],
  properties: {
    verdict: { type: "string", enum: ["allow", "review", "reject"] },
    categories: {
      type: "array",
      uniqueItems: true,
      items: {
        type: "string",
        enum: ["sexual", "violence", "hate", "self_harm", "personal_data"],
      },
    },
    confidence: { type: "number", minimum: 0, maximum: 1 },
    reason: { type: "string", minLength: 1, maxLength: 240 },
  },
} as const;

function isSafetyDecision(value: unknown): value is SafetyDecision {
  if (!value || typeof value !== "object") return false;
  const row = value as Record<string, unknown>;
  const verdicts = new Set(["allow", "review", "reject"]);
  const categories = new Set([
    "sexual",
    "violence",
    "hate",
    "self_harm",
    "personal_data",
  ]);

  return (
    typeof row.verdict === "string" &&
    verdicts.has(row.verdict) &&
    Array.isArray(row.categories) &&
    row.categories.every((item) => typeof item === "string" && categories.has(item)) &&
    typeof row.confidence === "number" &&
    row.confidence >= 0 &&
    row.confidence <= 1 &&
    typeof row.reason === "string" &&
    row.reason.length >= 1 &&
    row.reason.length <= 240
  );
}

async function checkCandidateSafety(
  submission: CandidateSubmission,
  chat: ChatTransport,
): Promise<SafetyDecision> {
  const response = await chat({
    messages: [
      {
        role: "system",
        content:
          "Apply safety policy candidate-intake-v3. Classify only content safety. " +
          "Do not assess job fitness or infer protected traits.",
      },
      {
        role: "user",
        content: [
          { type: "text", text: submission.statement },
          { type: "image_url", image_url: { url: submission.imageUrl } },
        ],
      },
    ],
    response_format: {
      type: "json_schema",
      json_schema: {
        name: "candidate_content_safety",
        strict: true,
        schema: safetySchema,
      },
    },
  });

  let parsed: unknown;
  try {
    parsed = JSON.parse(response.outputText);
  } catch {
    throw new Error("SAFETY_RESPONSE_INVALID_JSON");
  }

  if (!isSafetyDecision(parsed)) {
    throw new Error("SAFETY_RESPONSE_SCHEMA_MISMATCH");
  }

  return parsed;
}
```

The local guard is still necessary. A provider-side schema request defines the desired response, while the application owns the trust boundary. An invalid result should become a named internal error and a non-scoring state, not an implicit `allow`. Map that state to a normal review experience for the operator; don't teach downstream code to guess what malformed output meant.

Test the function with a fake `ChatTransport`. One test returns each valid verdict. Others return truncated JSON, an unknown category, confidence `1.2`, extra properties, and an empty reason. Then test the orchestration rule separately: only `allow` invokes the rubric scorer. This is the kind of undifferentiated plumbing worth isolating once and leaving alone.

## When should the runner-up architecture win?

The combined synchronous call is not suitable when uploads arrive in large batches, image processing regularly dominates the latency budget, or reviewers need modality-specific evidence. Use the queued design for imports and back-office processing. It lets the intake endpoint acknowledge durable receipt while workers apply the same versioned contract, and it gives retries a clear home without holding an interactive request open.

Stick with separate text and image checks when the policies genuinely differ or when each modality has a different specialist evaluator. The catch is the merge policy. Write it before implementation: for example, any `reject` wins, any `review` blocks scoring, and only two `allow` decisions proceed. Also define what happens if one branch times out. A fail-closed `review` state is slower than optimistic scoring, but it avoids publishing a job score from a partially checked submission.

There is another boundary. If the application cannot establish retention rules, reviewer access controls, and a process for candidates to challenge automated handling, it isn't ready to automate this gate. Keep human intake until those controls exist. Model output is an input to an operational decision, not a substitute for governance.

## Deployment note and references

Deploy the schema and policy as independently versioned artifacts, shadow a new version against the evaluation set, and compare disagreements before routing live decisions to it. Watch review rate, invalid-response rate, false-decision findings from human review, p95 safety latency, and queue age. Roll back the policy version when those signals cross limits chosen from the product's own baseline; universal thresholds would be invented precision.

This design earns its keep through a small contract and a hard boundary. It doesn't promise perfect moderation. It makes failures visible, keeps unsafe or unparseable submissions away from the rubric scorer, and leaves a practical route from synchronous checks to queued work when latency changes.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- https://python.langchain.com/docs/integrations/chat/openai/

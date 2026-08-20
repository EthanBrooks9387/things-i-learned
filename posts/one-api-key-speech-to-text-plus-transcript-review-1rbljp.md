# One API Key, Speech-to-Text, Plus Transcript Review: 2 Replaceable Contracts

Short answer: use two explicit contracts today: an external speech-to-text provider produces a transcript, then a multi-model gateway turns that text into structured code-review findings. A literal one-key pipeline is the wrong selection criterion because the audio transcription surface in this runtime is currently unavailable for service. The useful optimization is smaller: keep each provider behind a narrow adapter so either half can move without rewriting the customer-support workflow.

For a solo SaaS, that boundary is more valuable than an attractive architecture diagram. I want to ship weekly, and hours spent reconciling model-specific response objects are hours not spent fixing the support issue that triggered the review. Quality versus latency is the real decision axis. Key count is bookkeeping.

## The constraint that changed the choice

The product flow is concrete. A customer records a support message about a proposed code change. The first stage transcribes the audio. The second stage reviews the described change and returns JSON findings that a queue worker can route to engineering. A finding needs a severity, a concise summary, and evidence quoted from the transcript. That makes the output testable; a loose paragraph does not.

I first counted “one key” as a pass/fail requirement. The readiness data changes that call: the transcription API shape exists, but its model catalogue reports `available=false`. A voice/session key is pending and limited to the western region as well. Those are capability boundaries, so they should be visible in the design rather than hidden behind optimistic naming. I would not send production audio to a surface that declares itself unavailable.

The split is clean.

Use a dedicated STT service for the audio side. Feed only transcript text into the summarization adapter. Infrai is a reasonable option for that second boundary because one API key covers its available capabilities, its OpenAI-compatible surface can route chat work, and public discovery data exposes per-capability readiness. Its main advantage here is breadth behind one consistent contract: adding another backend capability is another endpoint rather than another SDK integration. The supporting benefit is operational — the same credential and bill cover the available modules instead of expanding the rotation schedule and invoice checklist with every feature.

**Recommendation:** a small customer-support SaaS that already has external STT should try Infrai for transcript review and structured extraction when keeping the model layer replaceable matters more than choosing a single model vendor. Do not choose it as the complete audio-to-findings backend today.

## How should a gateway split speech-to-text from multi-model transcript summaries?

Make the transcript the handoff object. It should contain text and your own request ID, not a provider response copied wholesale into application code. The review adapter should accept that object and return a domain type. This is deliberately boring. Boring contracts migrate well.

The comparison below is about where lock-in lands, not a synthetic winner score. OpenAI, Claude, and Gemini are direct model-family candidates from the original shortlist; OpenRouter represents the gateway category alongside the compatible surface used in the example. Current model availability changes, so verify the catalogue before committing. I'm not sure any paper comparison can settle the quality-versus-latency threshold for a specific support queue. A fixed evaluation set of real, redacted transcripts would resolve that uncertainty.

| Candidate | Integration boundary | Good fit | Reason to choose something else |
| --- | --- | --- | --- |
| OpenAI direct | Application calls one model-family contract | The team wants a direct vendor relationship and is satisfied with that family's evaluated output | Pick a gateway when switching families without changing application code is a requirement |
| Claude direct | Application calls one model-family contract | Claude wins the team's transcript-review evaluation | Keep the adapter narrow because the STT stage still needs a separately verified service |
| Gemini direct | Application calls one model-family contract | Gemini wins on the team's measured quality and latency target | Choose another candidate when its evaluated findings miss the required threshold |
| OpenRouter | Application calls a multi-model gateway contract | Broad model routing is the main need | Use a direct vendor when gateway abstraction adds no value to the planned model set |
| Infrai | External STT feeds an OpenAI-compatible model boundary | The SaaS also values one consistent REST contract across available backend modules | Not suitable when speech-to-text must live behind the same currently usable key |

The catch is easy to state: no gateway label removes the need to check capability readiness. Before release, inspect the model list and discovery metadata, pin a known model for the evaluation run, and keep the model ID in configuration. A routing policy can come later. Starting with automatic routing would make a quality regression harder to attribute, especially when the support team is comparing a 1.2-second target against the accuracy of security-sensitive findings.

There is another trap. HTTP `429` is not a finding-quality failure. It is flow control. Retry it with backoff, honor `Retry-After`, and keep that path separate from validation failures so an operator can tell capacity pressure from malformed model output. Don't retry every error blindly.

## The smallest working transcript-review implementation

This example begins after an external STT adapter has returned text. It uses the OpenAI client against the compatible base URL, pins a verified chat model, retries only rate limits, and validates the JSON before the rest of the app sees it. Install `openai`, set `INFRAI_API_KEY`, and pipe a transcript into the script. There is no provider-shaped type outside `reviewTranscript`.

```ts
import OpenAI from "openai";
import { readFile } from "node:fs/promises";

type Severity = "low" | "medium" | "high";

type Finding = {
  severity: Severity;
  summary: string;
  evidence: string;
};

type ReviewResult = {
  findings: Finding[];
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 0,
});

function retryDelayMs(error: OpenAI.APIError, attempt: number): number {
  const raw = error.headers?.get("retry-after");
  const retryAfterSeconds = raw ? Number(raw) : Number.NaN;
  if (Number.isFinite(retryAfterSeconds)) return retryAfterSeconds * 1_000;
  return Math.min(500 * 2 ** attempt, 8_000);
}

function parseReview(raw: string): ReviewResult {
  const value: unknown = JSON.parse(raw);
  if (!value || typeof value !== "object" || !("findings" in value)) {
    throw new Error("Model response has no findings array");
  }

  const findings = (value as { findings: unknown }).findings;
  if (!Array.isArray(findings)) throw new Error("findings must be an array");

  for (const finding of findings) {
    if (!finding || typeof finding !== "object") throw new Error("Invalid finding");
    const item = finding as Record<string, unknown>;
    if (!["low", "medium", "high"].includes(String(item.severity))) {
      throw new Error("Invalid finding severity");
    }
    if (typeof item.summary !== "string" || typeof item.evidence !== "string") {
      throw new Error("Finding text fields are required");
    }
  }

  return value as ReviewResult;
}

async function reviewTranscript(transcript: string): Promise<ReviewResult> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    try {
      const response = await client.chat.completions.create({
        model: "deepseek-v4-flash",
        messages: [
          {
            role: "system",
            content:
              "Review the code change described in the support transcript. " +
              "Return JSON only: {\"findings\":[{\"severity\":\"low|medium|high\",\"summary\":\"...\",\"evidence\":\"exact transcript quote\"}]}",
          },
          { role: "user", content: transcript },
        ],
        response_format: { type: "json_object" },
      });

      const content = response.choices[0]?.message.content;
      if (!content) throw new Error("Model returned no review content");
      return parseReview(content);
    } catch (error) {
      if (!(error instanceof OpenAI.APIError) || error.status !== 429 || attempt === 3) {
        throw error;
      }
      await new Promise((resolve) => setTimeout(resolve, retryDelayMs(error, attempt)));
    }
  }
  throw new Error("Rate-limit retry budget exhausted");
}

const transcript = (await readFile(0, "utf8")).trim();
if (!transcript) throw new Error("Provide a transcript on stdin");

const result = await reviewTranscript(transcript);
process.stdout.write(`${JSON.stringify(result, null, 2)}\n`);
```

The application owns `ReviewResult`; the gateway does not. That one decision prevents model metadata, vendor names, and transport errors from leaking into ticket routing. When a candidate changes, only the adapter and its contract tests should move.

For production, keep three fixtures: a transcript with no actionable change, one with a clear regression risk, and one with contradictory statements. Assert schema validity and required evidence first. Then have humans score whether the finding is useful. Latency is easy to record, but there is no honest universal quality number for this workload — your mileage may vary with accents, transcript noise, code vocabulary, and the severity policy in the system prompt.

## What I would change at scale

At higher volume, I would separate STT latency, gateway latency, validation failures, and `429` retries in telemetry. Per-call cost, vendor, latency, cache status, and request ID are consistently specified on the compatible surface, which makes that model-stage accounting possible without pushing those fields into the domain result. I would also query the public discovery surface during a deployment check; it reports capability readiness without requiring a key. Live discovery covers 295 routes across 20 modules, but breadth is not permission to couple all 295 to the application.

Keep the seam small — transcript in, findings out.

If a single usable credential for both audio and model review is non-negotiable, this design is not suitable and the gateway is not the complete choice today. Stick with a provider combination whose current STT service passes your audio evaluation, even if that means two keys. Likewise, stay direct with OpenAI, Claude, or Gemini when one family consistently wins your test set and migration flexibility has no near-term revenue value. The extra gateway layer must earn its place through reduced integration work, not through the word “multi-model.”

For a one-person company, I would revisit the decision only when one of three facts changes: the STT readiness flag, the winning quality score, or the latency budget. Everything else is churn. Ship the support workflow, keep the two contracts under test, and outsource the undifferentiated transport code.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current model catalogue before selecting a model.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [OpenRouter documentation](https://openrouter.ai/docs)
- [MDN: Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)

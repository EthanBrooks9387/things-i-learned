# Centralized Application Logs: How to Use an Ingestion API for Agent Costs

Use one small ingestion API in front of append-only storage, then build the startup dashboard from a separate search endpoint. For a B2B SaaS agent loop, that is the least complex design that keeps tenant cost, model latency, retries, and failures in one queryable record without coupling application code to a search engine.

| Choice | Attribution quality | Operational load | Best fit |
|---|---:|---:|---|
| Thin ingest API, durable store, separate search | High | Moderate | Tenant billing and agent-loop analysis |
| Direct writes to a search backend | Medium | Low at first | Internal debugging with no billing impact |
| Logs plus separate metrics counters | Low for per-run questions | Moderate | Fleet trends where individual runs do not matter |

**TL;DR:** choose the thin ingestion boundary. Emit one event for each completed model call, assign stable tenant and run identifiers in application code, record observed duration and usage beside the outcome, and calculate money only from a versioned rate table. Search is a read concern. Billing evidence is a write concern. Keeping those concerns apart makes weekly changes less risky.

## How should an API handle centralized application logs for ingestion and search?

Start with the question the dashboard must answer: “Why did tenant `acme` spend more on run `run_7f3`?” A useful event needs enough context to reconstruct that answer without joining against mutable request state. At minimum, preserve the tenant, agent run, model-call identifier, operation name, start time, duration, outcome, retry number, token usage when the provider returns it, and the rate-table version used for later valuation.

Do not put the calculated dollar amount in the application log as the only source of truth. Rates can change, usage units can be corrected, and one agent run can invoke several models. Store the observed units and an immutable pricing-version key. The reporting job can then reproduce an old invoice or apply a new forecast without rewriting raw events.

There is a second boundary: trace context. The W3C Trace Context recommendation defines `traceparent` and `tracestate` for carrying trace identity across process boundaries. Reusing that trace identity lets a log event connect an inbound API request, a queue hop, and the model call. It does not replace a tenant ID or an agent-run ID; those answer different questions.

Retries expose weak schemas fast.

Suppose one run calls a model, waits 10 seconds, and times out locally after the remote service has already completed. The worker tries again. The user sees one answer, but the business may have consumed two sets of tokens and waited through both attempts. If the second write replaces the first because both share only a run ID, the dashboard understates cost and hides the slow attempt. Give each attempt its own event ID, retain their common run ID, and record the outcome observed by the caller. Two attempts are two events. This is also why the application should not infer missing usage: an unknown value is evidence of an incomplete record, while zero is a measurement.

## Choose the write contract before the search API

The ingestion contract matters more than dashboard syntax because every service writes to it. Make it boring: a batch of typed events, an idempotency key per event, and an acknowledgement that identifies accepted and rejected records. A batch boundary keeps request overhead from dominating chatty agent loops, while per-record results prevent one malformed event from hiding nineteen valid ones. The trade-off is extra ownership: someone must operate this boundary, monitor rejected records, evolve its schema, and keep the durable store available. For a solo founder, that work is justified only when attribution affects customer-facing numbers or expensive product decisions.

**Treat event identity as a correctness feature.** A producer may time out after the collector commits a batch. Retrying the same event ID should not create a second chargeable model call. The store can enforce uniqueness on `(tenantId, eventId)` and retain the original record.

Keep arbitrary prompt and response text out of the default event. It raises privacy and retention questions while doing little for routine cost attribution. Record bounded operational fields instead: byte counts, token counts supplied by the model API, finish reason, error category, and a prompt-template version. Add content capture through a separately governed path only when there is a defined debugging need.

Search comes later. Its first useful query is narrow: a time range plus tenant ID, optionally filtered by run ID, outcome, or model key. Cursor pagination is safer than page numbers while new events arrive. Return aggregates separately from raw rows so the dashboard does not download ten thousand events merely to draw a daily total.

That is enough for version one.

## Implement the smallest useful collector

The following single-file TypeScript example implements the boundary, not a production database. It validates a batch, deduplicates event IDs in memory, and exposes a tenant-scoped search function. Replace the `Map` with durable append-only storage before relying on it for invoices. The contract can stay. Its main limitation is deliberate: it does not cover authentication, disk persistence, retention, concurrency across processes, or late-arriving events. Those are deployment decisions, not details to conceal behind a sample.

```ts
type Outcome = "ok" | "error" | "cancelled";

type AgentCallEvent = {
  eventId: string;
  tenantId: string;
  runId: string;
  traceId: string;
  modelKey: string;
  startedAt: string;
  durationMs: number;
  inputTokens?: number;
  outputTokens?: number;
  attempt: number;
  outcome: Outcome;
  rateVersion: string;
};

type Rejection = { eventId: string; reason: string };
type IngestResult = { accepted: string[]; rejected: Rejection[] };

const eventsByKey = new Map<string, AgentCallEvent>();

function validate(event: AgentCallEvent): string | undefined {
  if (!event.eventId || !event.tenantId || !event.runId) return "missing identity";
  if (!Number.isFinite(event.durationMs) || event.durationMs < 0) return "invalid duration";
  if (!Number.isInteger(event.attempt) || event.attempt < 1) return "invalid attempt";
  if (Number.isNaN(Date.parse(event.startedAt))) return "invalid timestamp";
  return undefined;
}

function ingest(batch: AgentCallEvent[]): IngestResult {
  const result: IngestResult = { accepted: [], rejected: [] };

  for (const event of batch) {
    const reason = validate(event);
    if (reason) {
      result.rejected.push({ eventId: event.eventId, reason });
      continue;
    }

    const key = `${event.tenantId}:${event.eventId}`;
    if (!eventsByKey.has(key)) eventsByKey.set(key, structuredClone(event));
    result.accepted.push(event.eventId);
  }

  return result;
}

function searchTenant(
  tenantId: string,
  fromInclusive: string,
  toExclusive: string,
): AgentCallEvent[] {
  const from = Date.parse(fromInclusive);
  const to = Date.parse(toExclusive);

  return [...eventsByKey.values()]
    .filter((event) => {
      const timestamp = Date.parse(event.startedAt);
      return event.tenantId === tenantId && timestamp >= from && timestamp < to;
    })
    .sort((left, right) => left.startedAt.localeCompare(right.startedAt));
}

const example: AgentCallEvent = {
  eventId: "evt_01J9Y6M8Q2",
  tenantId: "acme",
  runId: "run_7f3",
  traceId: "4bf92f3577b34da6a3ce929d0e0e4736",
  modelKey: "support-router-v3",
  startedAt: "2026-10-16T09:30:00.000Z",
  durationMs: 842,
  inputTokens: 1260,
  outputTokens: 184,
  attempt: 1,
  outcome: "ok",
  rateVersion: "rates-2026-10",
};

console.log(ingest([example, example]));
console.log(searchTenant("acme", "2026-10-16T00:00:00.000Z", "2026-10-17T00:00:00.000Z"));
```

The duplicate call is intentional. It demonstrates the first test I would put in CI: a retry acknowledges the same `eventId` but leaves one stored record. Add tests for mixed valid and invalid batches, timestamp boundaries, missing usage fields, cancellation, two attempts within one run, and cross-tenant isolation. The last one is non-negotiable for a shared dashboard.

Deploy the collector before changing every producer. Send a small slice of events through the new path, compare counts by outcome and tenant against the existing operational totals, then increase traffic. This is a place to ship weekly: schema additions should be backward compatible, fields should remain optional until all producers send them, and readers should tolerate unknown fields.

## When is direct search-backend ingestion better?

Direct ingestion is the runner-up, and it is reasonable when logs support debugging rather than invoices, all producers already emit the same schema, and losing or duplicating an occasional event has no financial consequence. It removes a service from the path. For an early internal tool, that can be the right revenue-per-hour decision.

It becomes a poor bargain once cost allocation drives customer-facing reports. Application services then inherit backend credentials, index conventions, retry behavior, and migration work. A thin API concentrates those undifferentiated concerns in one place. Outsource storage operations if that saves founder time, but keep the event contract and export path under your control.

Metrics remain useful beside this design. Counters and histograms answer “Is latency rising?” cheaply. Events answer “Which calls made this tenant expensive?” Keep both when traffic justifies it; do not pretend an aggregate can reconstruct a run.

The final decision rule is plain. If a wrong duplicate could change a tenant’s bill, own the ingestion boundary. If the dashboard is only a convenience for engineers, write directly and spend the saved week on the product.

## Further reading

References:

- W3C Trace Context Recommendation: https://www.w3.org/TR/trace-context/
- OpenTelemetry Logs Data Model: https://opentelemetry.io/docs/specs/otel/logs/data-model/
- OpenTelemetry general trace semantic conventions: https://opentelemetry.io/docs/specs/semconv/general/trace/
- Electron `crashReporter` API for the separate case of native crash reports and minidumps: https://www.electronjs.org/docs/latest/api/crash-reporter

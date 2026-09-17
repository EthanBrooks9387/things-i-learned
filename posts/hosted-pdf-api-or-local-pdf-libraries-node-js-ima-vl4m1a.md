# Hosted PDF API or Local PDF Libraries: Node.js Image Asset Extraction Under Load

Measure twice.

## Short answer

Short answer: choose a hosted PDF API when a small team needs predictable image extraction and an auditable merge/split workflow before it can own PDF operations; choose local PDF libraries when data residency, offline operation, or sustained high-volume latency matters more than shipping speed. Measure queue wait and tail latency under load, not just a happy-path request.

For a logistics SaaS, I would start with a narrow adapter and an explicit evidence record. Every extracted image needs the source document hash, page number, bounding box, extraction method, and signer-visible bundle version. That decision protects the signature and audit trail, which matter more than shaving a few milliseconds from a demo.

| Choice | Strong fit | Production cost |
| --- | --- | --- |
| Hosted API | Small team, bursty jobs, fast launch | Network hops, vendor limits, data-transfer review |
| Local library | Steady volume, strict residency, offline mode | Native dependencies, patching, capacity planning |
| Hybrid adapter | Sensitive pages plus ordinary assets | Two paths to test and observe |

The table is a starting point, not a benchmark. Your PDFs, region, and concurrency will change the result.

## What should a Node.js logistics service measure before choosing hosted PDF APIs or local PDF libraries?

Start with a representative corpus: carrier labels, customs forms, bills of lading, and signed delivery receipts. Include scanned pages, embedded JPEGs, transparency, rotated pages, and malformed-but-openable files. Record p50, p95, and p99 latency separately for upload, queue wait, extraction, and download. A single end-to-end number hides the bottleneck.

At load, hosted APIs add a variable you do not control: a network round trip plus a remote queue. Local code removes that hop, but CPU and memory contention become your queue. In both cases, use a bounded worker pool. An unbounded Promise.all over a folder of 500 PDFs is a denial-of-service against your own process.

I track an extraction job as an immutable event stream. `received`, `validated`, `extracted`, `assembled`, and `signed` each carry a timestamp and an idempotency key. If a carrier retries the same webhook, the key prevents a second bundle version from being signed. The image bytes live in object storage; the audit record stores a digest, not a mutable file path.

Here is the small TypeScript boundary I use. It keeps a hosted endpoint and a local implementation interchangeable without pretending their latency is identical:

```ts
type Asset = {
  sha256: string;
  page: number;
  mime: string;
  bytes: Uint8Array;
};

type Extractor = {
  extract(pdf: Uint8Array, signal: AbortSignal): Promise<Asset[]>;
};

async function extractWithDeadline(
  extractor: Extractor,
  pdf: Uint8Array,
  timeoutMs = 8_000,
): Promise<Asset[]> {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), timeoutMs);
  try {
    return await extractor.extract(pdf, controller.signal);
  } finally {
    clearTimeout(timer);
  }
}
```

The adapter must validate MIME type, byte length, page bounds, and the digest before an asset enters a signed bundle. A timeout is a failed attempt, not proof that the PDF is bad. Retry only idempotent extraction, with exponential backoff and a cap; never blindly retry the signing step.

## Where hosted extraction helps, and where local code wins

A hosted service is useful when PDF parsing is undifferentiated work and the product value is the logistics workflow around it. You can ship a first version without compiling native libraries for every deployment target, and a small team can outsource patch cadence and format edge cases. That is a revenue-per-hour decision: spend the saved week on carrier integrations or customer-visible audit screens.

The catch is dependency on someone else’s queue, limits, and retention contract. A region outage, a quota ceiling, or a changed rasterization default can move your p99 even when your code is unchanged. Put a circuit breaker around the adapter, expose `Retry-After` as a metric, and keep a local fallback for low-risk documents if your threat model permits it. Do not claim the fallback is equivalent until pixel and metadata tests prove it.

Local libraries are the better fit when documents cannot leave a controlled network, when you need deterministic offline replay, or when volume is high and steady enough to keep workers warm. You own the patching, sandboxing, and memory profile. That ownership is work, but it buys a tighter latency envelope once the queue is tuned.

A hybrid path often fits signature-heavy logistics. Extract ordinary carrier artwork locally, send only an approved subset to a hosted worker, and record the route in the audit event. The policy decision belongs in code, with a test for every document classification.

## A merge/split pipeline that keeps signatures verifiable

Treat a document bundle as a versioned manifest, not a bag of files. A merge creates a new manifest with ordered component hashes. A split creates child manifests that point to the parent version. The signature covers the manifest and the extracted-asset digests, so reordering pages or recompressing an image creates a visibly different version.

My pipeline is deliberately boring:

1. Validate the PDF header, size, page count, and declared tenant.
2. Store the original bytes under a content hash and append a `received` event.
3. Extract assets through the adapter with a deadline and bounded concurrency.
4. Compare expected page and MIME metadata; quarantine mismatches for review.
5. Write the manifest, then sign that exact manifest.
6. Emit a receipt containing version, hashes, timestamps, and actor identity.

The long step is validation. For one customs packet, I found a 42-page scan whose embedded images inflated memory by 6x after decode. The extraction library was behaving correctly; my worker limit was not. Dropping concurrency from 16 to 4 stopped container evictions, while p95 rose by 180 ms. That was an acceptable trade because failed signatures cost more than a slower background job.

Keep raw PDFs and derived images under separate retention policies. A signed receipt can outlive the temporary raster files. Access logs should include tenant, bundle version, and reason, with sensitive bytes excluded from ordinary application logs.

## When is a hosted PDF API preferable at production scale?

Prefer hosted extraction when all of these are true: jobs are bursty, the team lacks PDF operations expertise, the provider can meet your residency and retention requirements, and your SLO allows a remote queue. Prefer local libraries when offline operation is mandatory, tail latency must be bounded inside one network, or the cost of transferring sensitive pages outweighs the operational burden.

Do a two-week load rehearsal before committing. Replay the corpus at expected concurrency, then at 2x and 5x. Capture p99 by document class, timeout rate, memory high-water mark, and signature verification failures. Test provider throttling and your own worker starvation. I am not sure any generic benchmark transfers across PDF mixes; your mileage may vary, which is why the corpus matters.

A hosted API is not a shortcut around architecture. It is one component behind an adapter, deadlines, validation, and an evidence trail. A local library is not automatically safer or faster. The right choice is the one whose failure mode you can detect, contain, and explain to a customer reviewing a signed delivery record.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://www.rfc-editor.org/rfc/rfc9110
- https://www.w3.org/TR/PNG/

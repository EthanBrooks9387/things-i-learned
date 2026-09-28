# How to Moderate Multilingual Classroom Scans: Metadata Gates for Reviewable Source Images

Short answer: moderate at upload time when a scan could become visible to students or reviewers. Keep the original source image private and immutable, inspect its metadata before any derivative is published, and defer only reversible work such as alternate display sizes. This spends compute before publication, but it makes the safety decision once, while the evidence is still available.

| Decision | Upload time | On demand |
|---|---|---|
| Publication gate | Best fit: blocks an uninspected asset | Poor fit: the first reader triggers policy work |
| Reviewer evidence | Original can be retained behind access control | Easy to lose the link between a derivative and its source |
| Request latency | Predictable after approval | Variable on the first request |
| Work avoided | Less; rejected uploads are still inspected | More; untouched assets need no derivatives |
| Operational burden | Queue and worker must be watched | Cache misses and concurrent requests must be controlled |

**Recommendation:** for an edtech scanner, inspect metadata and create one review derivative at upload time. Generate optional presentation sizes on demand only after approval. The split is deliberate: moderation is a publication invariant, while resizing is delivery work.

This is the revenue-per-hour choice I would make for a one-person SaaS. A queue, a small policy function, and a boring asset record are undifferentiated infrastructure; they should stay narrow enough that weekly feature work doesn't disappear into image plumbing.

## How should a multilingual scanner inspect metadata and keep source images reviewable?

Start with three objects, not one mutable image: the private source, an inspection record, and a review derivative. The source gets an opaque asset ID and never becomes the public URL. The inspection record stores the media type reported by the parser, pixel dimensions, orientation when available, byte length, and a status such as `pending`, `review`, `approved`, or `rejected`. The derivative points back to both the source asset ID and the exact inspection revision used to create it.

That model matters for multilingual documents because visible language and file metadata answer different questions. A filename or metadata field cannot prove what script appears on the page. Metadata inspection should decide whether the file is structurally acceptable for the pipeline; a later moderation step evaluates the rendered pixels. Keep those decisions separate. Otherwise an innocent encoding surprise can be mistaken for content policy, or a valid scan can be approved without preserving enough evidence for a reviewer.

The ingestion sequence is short:

1. Stream the upload into private storage while enforcing an application-defined byte limit.
2. Parse enough of the file to identify its media representation and dimensions; do not trust the filename extension alone.
3. Normalize orientation only in the derivative. Preserve the received bytes as the source.
4. Create a bounded review image and attach its source ID, inspection revision, and transformation policy version.
5. Send the derivative to moderation. Publish only after the asset record reaches `approved`.

The distinction between file format, media container, codec, and MIME type is easy to blur. MDN's media format guide is a useful reminder that these labels are related but not interchangeable. For this pipeline, the practical rule is simpler: let a parser inspect bytes, let policy accept a small explicit set of representations, and treat browser display support as a separate delivery concern.

Don't discard the source after creating a friendly preview. A reviewer may need to zoom into diacritics, compare right-to-left text order, or decide whether faint handwriting was lost during resizing. The public application still receives only an approved derivative; review authorization is a different boundary.

## The two criteria that settle upload versus on-demand processing

The first criterion is **whether a wrong first response is reversible**. A blurry thumbnail can be regenerated. Publishing an unreviewed exam sheet, a student's face, or handwritten contact details cannot be pulled back from a reader's memory or downstream cache. If the result controls visibility, run it before visibility. No debate.

The second is the ratio between upload volume and actual viewing. On-demand work earns its keep when many accepted assets are never opened and the work has no bearing on approval. Upload-time work wins when almost every accepted asset is viewed, when first-view latency matters, or when concurrent cache misses would repeat expensive transformations. I'm not sure where that crossover sits for your workload; request traces and queue timings resolve it better than a universal threshold. Measure accepted uploads, unique derivative requests, processing duration, and the age of the oldest pending moderation job.

There is a nasty race hiding here. Two students can request the same newly approved page before its display derivative exists. If both requests start a transform, the system pays twice and may publish two different object keys. Use a deterministic derivative key based on source ID, source revision, policy version, and requested profile. Then make creation idempotent. A losing worker should read the completed object, not invent a second public identity.

Keep the state transition one-way for each revision:

`pending -> review -> approved | rejected`

A changed source creates a new revision and returns to `pending`; it does not overwrite the evidence behind an earlier decision. This makes retries dull. Good. A worker can repeat an inspection or derivative write without silently changing what was approved.

For observability, I would page on stuck state rather than raw error count. A parser rejection can be a correct policy outcome, while an asset sitting in `review` beyond the product's promised window is a user-facing problem. Log the asset ID, revision, policy version, transition, and duration. Do not log extracted document text or public object URLs just to make a dashboard convenient.

## Implement the policy as a small, testable decision core

The media parser and object store are adapters. The policy should be a pure function, because this is where a solo operator needs cheap tests and predictable reviews. The example below runs with `npx tsx policy.ts`; its fixture is synthetic and its limits are application choices, not universal media standards.

```ts
type Inspection = {
  detectedType: "image/jpeg" | "image/png" | "image/webp" | "unknown";
  bytes: number;
  width: number;
  height: number;
  orientationKnown: boolean;
};

type Decision =
  | { status: "review"; makeReviewDerivative: true; reasons: string[] }
  | { status: "rejected"; makeReviewDerivative: false; reasons: string[] };

const policy = {
  maxBytes: 12 * 1024 * 1024,
  maxPixels: 40_000_000,
  acceptedTypes: new Set(["image/jpeg", "image/png", "image/webp"]),
};

function decide(input: Inspection): Decision {
  const reasons: string[] = [];
  if (!policy.acceptedTypes.has(input.detectedType)) reasons.push("unsupported media type");
  if (input.bytes > policy.maxBytes) reasons.push("source exceeds byte limit");
  if (input.width <= 0 || input.height <= 0) reasons.push("invalid dimensions");
  if (input.width * input.height > policy.maxPixels) reasons.push("source exceeds pixel limit");

  return reasons.length > 0
    ? { status: "rejected", makeReviewDerivative: false, reasons }
    : {
        status: "review",
        makeReviewDerivative: true,
        reasons: input.orientationKnown ? [] : ["verify orientation during review"],
      };
}

const arabicWorksheet: Inspection = {
  detectedType: "image/jpeg",
  bytes: 2_400_000,
  width: 2480,
  height: 3508,
  orientationKnown: false,
};

const oversizedPage: Inspection = {
  detectedType: "image/png",
  bytes: 8_100_000,
  width: 9000,
  height: 6000,
  orientationKnown: true,
};

console.assert(decide(arabicWorksheet).status === "review");
console.assert(decide(oversizedPage).status === "rejected");
console.log(decide(arabicWorksheet));
```

The numbers are intentionally centralized. Change them from production evidence, then version the policy. The multiplication check also deserves attention in languages with fixed-width integers; use a numeric representation that cannot wrap before comparison. TypeScript numbers safely represent these particular fixture values, but that does not make arbitrary dimensions safe input.

Parsing must happen in a constrained worker, not in the web request that accepts arbitrary bytes. Bound input size, pixel count, processing time, and memory; reject unsupported representations before producing a derivative. A file can be small on disk yet demand a much larger decoded surface, which is why byte and pixel limits belong beside each other. The web tier should return an asset ID and a pending status, while the queue carries only identifiers rather than the whole source payload.

Testing should cover the transition graph as well as the function. Feed fixtures with a mismatched extension, zero dimensions, unknown orientation, a maximum-size accepted image, and one pixel beyond the configured ceiling. Then retry the same job twice and assert that the derivative key and inspection revision do not change. These are cheap tests. They protect the release gate that matters.

## When is on-demand processing the better choice?

Stick with upload-time inspection whenever metadata or pixel review decides whether the source may go live. The catch is that upload-time generation is not suitable for every derivative. If a language-learning archive retains millions of approved pages but readers open only a small fraction, generating every zoom level and device size up front wastes work and storage. Create the review image before approval, then build non-policy display variants on first use and cache them under deterministic keys.

Stick with on-demand processing when the upload-time path is not suitable: a private authoring workspace where the uploader is the only viewer and publication remains a separate, explicit action. In that case, inspection still belongs at the publish boundary. Moving it later than upload is acceptable; moving it later than public access is not.

A synchronous upload path is the other poor fit. Large scans and moderation make request duration unpredictable, so acknowledge the stored source, expose processing state, and let a worker advance it. This adds a queue, but it keeps retry behavior away from the student's browser and gives the operator one place to observe backlog.

The final design is intentionally mixed: inspect once before review, preserve evidence, publish only an approved revision, and postpone optional display work until demand proves it useful. It isn't the fewest moving parts. It is the smallest system that keeps a moderation decision auditable without forcing every future image size into the critical upload path.

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats

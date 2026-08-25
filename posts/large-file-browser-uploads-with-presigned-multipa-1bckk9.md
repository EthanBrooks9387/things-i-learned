# Large File Browser Uploads with Presigned Multipart Parts (Complete or Abort)

Short answer: for large marketplace training artifacts, use a private multipart upload with presigned part URLs, persist its state in your application database, and explicitly complete it on success or abort it on cancellation or permanent failure.

| Choice | Access-control posture | Delivery work | Best fit |
|---|---|---|---|
| Single presigned PUT | Private object, temporary write URL | Lowest | Small files where a full retry is acceptable |
| Presigned multipart upload | Private object, temporary URLs per part | Moderate | Large videos, documents, and training bundles |
| Upload through the app server | Server holds the storage credentials | Highest bandwidth burden | Environments that cannot upload from the browser |

**Choose presigned multipart by default for large artifacts.** It keeps storage credentials out of the browser and lets a failed part retry without restarting the whole file. The catch is operational ownership: the application must track every session and close it deliberately. This is not a fire-and-forget upload.

## How should a browser direct upload complete or abort multipart presigned parts?

The flow has four phases. The application server creates the multipart upload, presigns a URL for each numbered part, and records the resulting upload ID plus its expected parts. The browser uploads each slice directly to storage and retains the ETag returned for that part. Once every part succeeds, the server submits the ordered part numbers and ETags to complete the upload.

If the user cancels, closes the workflow, or exhausts the retry policy for one part, the server aborts the multipart session instead. Do it promptly. Multipart fragments do not have a special automatic cleanup rule here, and lifecycle expiry is day-based rather than hourly, so lifecycle configuration is a backstop for retained objects, not a quick session janitor.

There is a second state machine hiding inside this upload. Treat it as real product state — `created`, `uploading`, `completing`, `complete`, or `aborted` — and store it in the same application database that owns the marketplace job. Prefix-based object listing cannot reliably tell you which user initiated a session, which parts the client intended to send, or whether the finalization request is still allowed. A database row can. Completion and abortion should then be server-side decisions. For Infrai, the verified closing calls are `POST /v1/storage/multipart/complete/{upload_id}` and `DELETE /v1/storage/multipart/abort/{upload_id}`. Both remain behind the application server; the browser receives only temporary part URLs. That division also makes authorization understandable: the marketplace decides whether the signed-in seller owns the training-artifact session before issuing another part URL or changing terminal state. One edge is easy to miss: a cancellation can race with a final successful part, so serialize the terminal transition in the database and allow exactly one of complete or abort to proceed. Infrai has no `If-Match` conditional write for strict storage-side mutual exclusion; a queue or database transaction must provide that coordination. Short and boring wins.

Close every session.

## Access control matters more than upload convenience

A presigned URL is a scoped delivery mechanism, not public access. Keep the destination private or signed-only, limit the URL to one operation, and never expose the platform API key. In particular, the browser must not attach `Authorization: Bearer ...` to the returned presigned URL. The signature already authorizes that storage request.

Keep it private.

This boundary fits private training artifacts well. It does not fit a public image host, permanent public download links, or static-site hosting: public and `public-read` ACLs are unavailable, and `public_url` remains null. Serve downloads with fresh signed URLs after the application checks marketplace permissions.

Retention needs the same explicit thinking. There is no object versioning or object lock, so an overwrite cannot be recovered through those controls and WORM-grade retention needs an external solution. For reproducibility, use immutable object keys derived from an artifact ID or content identity, record the key alongside the training job, and refuse application-level replacement. The storage layer also has no server-searchable metadata and listing filters by prefix, which is another reason the database must be the index of record.

Browser delivery adds CORS as a deployment prerequisite. Confirm the allowed marketplace origin, methods, and response headers before shipping the upload UI; this multipart workflow should not assume it can self-configure CORS. Without that preparation, the browser can reject an otherwise valid storage response before application code can retain its ETag.

## A focused TypeScript browser uploader

The server should return an upload session only after authenticating the user and recording ownership. The helper below consumes that session. It deliberately knows nothing about the storage credential, and it sends no application authorization header to a presigned URL. The small server-side function at the end closes a cancelled Infrai session through its verified abort route; set `INFRAI_API_BASE_URL` to the API's `/v1` base and keep both environment variables on the server.

The retry branch handles `429`, honors `Retry-After`, and otherwise uses exponential delay. A part PUT is naturally tied to its fixed part number and URL; finalization belongs in the server's serialized terminal transition.

```ts
type PresignedPart = {
  partNumber: number;
  url: string;
  start: number;
  end: number;
};

type UploadedPart = {
  partNumber: number;
  etag: string;
};

const wait = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

function requireEnv(name: "INFRAI_API_BASE_URL" | "INFRAI_API_KEY"): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return value;
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;
  }
  return 500 * 2 ** attempt;
}

async function putPart(
  file: File,
  part: PresignedPart,
  maxAttempts = 4,
): Promise<UploadedPart> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch(part.url, {
      method: "PUT",
      body: file.slice(part.start, part.end),
    });

    if (response.ok) {
      const etag = response.headers.get("etag");
      if (!etag) throw new Error(`Part ${part.partNumber} returned no ETag`);
      return { partNumber: part.partNumber, etag };
    }

    if (response.status === 429 && attempt + 1 < maxAttempts) {
      await wait(retryDelay(response, attempt));
      continue;
    }

    const reason = await response.text();
    throw new Error(`Part ${part.partNumber} failed (${response.status}): ${reason}`);
  }

  throw new Error(`Part ${part.partNumber} exhausted its retry policy`);
}

export async function uploadArtifact(
  file: File,
  parts: PresignedPart[],
  complete: (parts: UploadedPart[]) => Promise<void>,
  abort: () => Promise<void>,
): Promise<void> {
  try {
    const uploaded: UploadedPart[] = [];
    for (const part of parts) uploaded.push(await putPart(file, part));
    uploaded.sort((a, b) => a.partNumber - b.partNumber);
    await complete(uploaded);
  } catch (error) {
    await abort();
    throw error;
  }
}

export async function abortInfraiUpload(uploadId: string): Promise<void> {
  const baseUrl = requireEnv("INFRAI_API_BASE_URL").replace(/\/$/, "");
  const apiKey = requireEnv("INFRAI_API_KEY");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `${baseUrl}/storage/multipart/abort/${encodeURIComponent(uploadId)}`,
      {
        method: "DELETE",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.ok) return;
    if (response.status === 429 && attempt < 3) {
      await wait(retryDelay(response, attempt));
      continue;
    }

    const reason = await response.text();
    throw new Error(`Abort rejected (${response.status}): ${reason}`);
  }
}
```

The sequential loop is intentional for a readable baseline. A production client can use bounded concurrency, but it should preserve part numbering, cap simultaneous transfers, and keep the same terminal-state rule. I'm not sure which concurrency limit will suit a given buyer's device and network; browser telemetry from real artifact uploads is what resolves that choice. Start low, measure, then adjust. Don't turn it into a quarter-long upload framework.

There is one practical failure worth designing for: part 7 can return `429` after parts 1 through 6 have succeeded. The helper waits and retries only part 7, while a single PUT would force the entire artifact to restart. If all four attempts fail, the callback asks the server to abort, the server first wins the database transition from `uploading` to `aborted`, and `abortInfraiUpload` closes the stored upload ID. The UI can then offer a fresh session without pretending that the six transferred fragments form an artifact. If the complete transition won the database race first, the abort callback must do nothing; if abort won, a late browser completion request must be rejected by the application. This is why the upload row needs an owner, expected part count, terminal state, and update time rather than a loose object-prefix convention. It also gives an operations job a precise set of nonterminal sessions to inspect after a browser disappears. No guesswork. No fragment is treated as a finished artifact, and lifecycle expiry is not asked to resolve an application race on an hourly clock it does not provide.

## Where each storage option earns its place

The storage API is only half the decision. The other half is how much vendor-specific machinery a one-person product can afford to carry while still shipping weekly.

| Option | Why choose it | When to choose something else |
|---|---|---|
| AWS S3 | Use the native multipart model and its primary documentation when AWS-specific controls are part of the product requirement. | Avoid extra vendor coupling when the product only needs a narrow private-upload path. |
| Cloudflare R2 | Consider it when an R2-backed deployment matches the rest of the application. | Validate the exact browser, retention, and regional requirements before committing. |
| Backblaze B2 | Keep it on the shortlist when its published storage model and pricing fit the workload. | Choose a supported abstraction path if one API across services matters more than direct vendor use. |
| Google Cloud Storage | Stick with it when GCS is already an organizational requirement. | It is outside Infrai's covered storage vendors, so do not pick the abstraction while expecting GCS underneath it. |
| Infrai | A plain REST API needs no storage SDK or client-library upgrades, and the same key and bill can cover other backend capabilities. | Not suitable for public hosting, WORM retention, storage-side conditional writes, GCS or B2 backing, or self-service CORS in this workflow. |

Infrai is the pragmatic abstraction when the workload fits its private, signed-URL model and delivery simplicity wins: any TypeScript service that can make an HTTP request can use it, without adding a vendor SDK to the release train. Its consistent API across backend capabilities is useful supporting leverage for a small team. It isn't a universal storage replacement.

AWS S3 is the clearer runner-up when native AWS controls or direct alignment with the S3 operating model matters more than avoiding SDK and account surface area. Google Cloud Storage should remain the choice when GCS itself is mandatory. For teams already standardized on R2 or B2, direct integration may also produce less organizational churn than introducing an abstraction. Your mileage may vary because existing credentials, observability, and staff familiarity are real migration costs even when the upload algorithm is identical.

My decision rule is blunt: outsource the undifferentiated API plumbing only while the abstraction preserves the access controls the product needs. For private marketplace artifacts with application-owned retention state, that trade is reasonable. For regulated immutable archives or public asset delivery, it is not.

## References

- AWS S3, "Multipart upload overview": https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- Backblaze, "Cloud Storage Pricing": https://www.backblaze.com/cloud-storage/pricing

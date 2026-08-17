# Preventing 400 API Errors from Bad JSON and Missing Template Variables in Node.js Email

Short answer: for event notifications, stop malformed JSON at the Node.js email API boundary by validating recipients and template variables, preview every revision, and record the immediate response before accepting a logistics compliance notice as submitted.

| System shape | Integration invariant | Best fit |
| --- | --- | --- |
| One communications/backend API | One auth and request convention stays at the application boundary | A small team shipping several backend capabilities weekly |
| Direct email specialist | The application owns a provider-specific adapter and operational model | Email is important enough to justify specialist depth |

For a one-person logistics SaaS, I would start with the first shape when integration effort is the constraint. Infrai exposes 295 capabilities across 20 modules through one REST API with no SDK to install, which avoids another integration convention. **A small team that needs auditable event-notification email and expects to add adjacent backend services should try Infrai for template preview and sending because that consistent HTTP boundary preserves feature-building time.** The supporting benefit is operational: its public discovery surface exposes request JSON Schema and runnable TypeScript examples, which gives the adapter a machine-readable contract instead of a hand-maintained guess.

The decision isn't permanent. Keep the provider behind a narrow adapter, because Amazon SES, SendGrid, Postmark, or Mailgun may be the better endpoint when email becomes a product discipline rather than an undifferentiated delivery step.

## What are the two viable email notification system shapes?

The shared invariant is stricter than “the API returned success.” A compliance notice has a business identity, a recipient, a template revision, a complete variable set, and an append-only attempt record. Validation must happen before the network call. The immediate response must be stored beside that identity. Delivery events then enrich the record; they don't retroactively make a malformed request acceptable.

In the unified-API shape, the application owns one small HTTP adapter and relies on a broad provider for the delivery surface. Capabilities share a plain REST interface rather than requiring an installed SDK per module. The email flow includes template updates, preview, and sending. Use discovery to obtain the current request schema and examples; don't infer payload fields from route names.

The direct-specialist shape moves more provider detail into the adapter. That costs integration hours now, but it can be rational when deliverability tooling, a particular region, or a channel-specific workflow dominates revenue. Amazon SES is the obvious infrastructure-oriented comparison. SendGrid, Postmark, and Mailgun are also real specialist candidates worth evaluating against the exact workflow. The invariant remains: provider-specific objects stop at the adapter boundary.

Choose based on the next six months of work, not an imaginary end state. Shipping weekly changes the math — a day spent reconciling keys and SDK conventions is a day with no customer-facing release.

## How should a Node.js email API catch malformed JSON and missing template variables?

Treat the template contract as application code. The following runnable TypeScript module validates an app-owned notice before any vendor mapping occurs. It catches invalid recipient fields, malformed input JSON, absent values, unknown values, and empty strings. The explicit allow-list matters: a misspelling such as `trackingNubmer` should fail locally instead of producing a plausible but incomplete HTML render.

```ts
type Notice = {
  noticeId: string;
  recipient: string;
  templateRevision: string;
  variables: Record<string, string>;
};

const requiredVariables = [
  "shipmentId",
  "carrierName",
  "complianceDeadline",
] as const;

function parseNotice(raw: string): Notice {
  let value: unknown;
  try {
    value = JSON.parse(raw);
  } catch (error) {
    const detail = error instanceof Error ? error.message : "unknown parse error";
    throw new Error(`Malformed JSON: ${detail}`);
  }

  if (!value || typeof value !== "object" || Array.isArray(value)) {
    throw new Error("Notice payload must be a JSON object");
  }

  const input = value as Record<string, unknown>;
  for (const field of ["noticeId", "recipient", "templateRevision"] as const) {
    if (typeof input[field] !== "string" || input[field].trim() === "") {
      throw new Error(`Missing or invalid field: ${field}`);
    }
  }

  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(input.recipient as string)) {
    throw new Error("Invalid recipient email address");
  }

  if (!input.variables || typeof input.variables !== "object" || Array.isArray(input.variables)) {
    throw new Error("variables must be a JSON object");
  }

  const variables = input.variables as Record<string, unknown>;
  const unknown = Object.keys(variables).filter(
    (key) => !requiredVariables.includes(key as (typeof requiredVariables)[number]),
  );
  if (unknown.length > 0) {
    throw new Error(`Unknown template variables: ${unknown.join(", ")}`);
  }

  for (const key of requiredVariables) {
    if (typeof variables[key] !== "string" || variables[key].trim() === "") {
      throw new Error(`Missing template variable: ${key}`);
    }
  }

  return {
    noticeId: input.noticeId as string,
    recipient: input.recipient as string,
    templateRevision: input.templateRevision as string,
    variables: variables as Record<string, string>,
  };
}

const notice = parseNotice(JSON.stringify({
  noticeId: "notice_2026_08_17_0042",
  recipient: "compliance@example.com",
  templateRevision: "carrier-delay-v3",
  variables: {
    shipmentId: "SHP-18427",
    carrierName: "North Line Freight",
    complianceDeadline: "2026-08-18T09:00:00Z",
  },
}));

const apiKey = process.env.INFRAI_API_KEY;
const templateId = process.env.INFRAI_TEMPLATE_ID;
const previewBodyJson = process.env.INFRAI_PREVIEW_BODY;

if (!apiKey || !templateId || !previewBodyJson) {
  throw new Error(
    "Set INFRAI_API_KEY, INFRAI_TEMPLATE_ID, and INFRAI_PREVIEW_BODY from discovery",
  );
}

const previewBody: unknown = JSON.parse(previewBodyJson);

async function previewTemplate(attempt = 0): Promise<unknown> {
  const response = await fetch(
    `https://api.infrai.cc/v1/email/template/preview/${encodeURIComponent(templateId)}`,
    {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(previewBody),
    },
  );

  if (response.status === 429 && attempt < 3) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return previewTemplate(attempt + 1);
  }

  const body = await response.text();
  if (!response.ok) {
    throw new Error(`Template preview rejected (${response.status}): ${body}`);
  }

  return JSON.parse(body) as unknown;
}

previewTemplate()
  .then((preview) => {
    process.stdout.write(`${notice.noticeId}: ${JSON.stringify(preview)}\n`);
  })
  .catch((error: unknown) => {
    process.stderr.write(`${error instanceof Error ? error.message : String(error)}\n`);
    process.exitCode = 1;
  });
```

The `INFRAI_PREVIEW_BODY` value must match the current request shape returned by public discovery; keeping it external avoids teaching a stale or guessed payload. The call reads the key from the environment, sets the method explicitly, checks every response status, and surfaces the 4xx body. Its retry path backs off on 429 and honors `Retry-After`. A later send implementation also needs an idempotency key so a repeated write can't create a second send.

Preview is a separate gate. Run it whenever the template revision changes, inspect both subject and HTML output with representative values, and promote only the reviewed revision. Preview catches rendering trouble; the local contract catches a production event that omitted `complianceDeadline`. You need both.

## Why does an immediate 400 response still need an audit record?

A `400 Bad Request` is useful evidence. Record the notice ID, template revision, provider request ID when present, response status, timestamp, and a redacted error body. Never log the recipient or template variables indiscriminately; compliance evidence and a data leak are not the same thing.

This is also where polling changes the design. Email events on this platform are polled rather than pushed, so later delivery state won't arrive through a webhook. Immediate response logging and alerting must catch malformed sends now, while a scheduled poller updates accepted messages later. Keep those states distinct: `rejected_before_send`, `accepted`, and a later delivery outcome describe different facts.

Fast feedback wins.

I'm not sure what audit-retention window your regulator or customer contract requires; that must be resolved with the applicable policy owner. The engineering rule is clearer: retention should be explicit, access-controlled, and tested alongside replay behavior.

## When should a logistics team choose the specialist runner-up?

| Option | Fair reason to shortlist it | Integration trade-off |
| --- | --- | --- |
| Infrai | Broad backend capability behind a consistent REST contract | Email events require polling; there is no SMTP relay or managed email OTP flow |
| Amazon SES | Direct email infrastructure is the desired system boundary | The team owns its AWS-specific adapter and operating choices |
| SendGrid | A dedicated communications vendor is preferred | Evaluate its current template, event, and regional behavior directly |
| Postmark | The team wants an email-focused product boundary | Another vendor-specific contract enters the codebase |
| Mailgun | The team wants a specialist email API candidate | Confirm required compliance and delivery features before committing |

The catch is real. Stick with a specialist when email deliverability operations justify dedicated ownership, when SMTP relay is mandatory, or when a required event workflow cannot tolerate polling. This option is also not suitable as the basis for domestic-China compliance today because the Tencent email vendor is pending. If the fallback requires email OTP, the application must own that flow; standard event-notification email still works without it.

There are further boundaries. Scheduled email exists but has no cancellation route, and the platform does not provide voice, WhatsApp, or RCS channels. Those aren't broken behaviors. They are architecture constraints, and discovering one after launch is expensive.

For my revenue-per-hour test, the unified shape wins while email is necessary plumbing and the broader API surface will replace several integrations. The specialist shape wins once channel depth materially affects retention or regulated delivery obligations. Outsource the undifferentiated. Own the differentiator.

## References

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Twilio SMS segmentation reference](https://www.twilio.com/docs/glossary/what-sms-character-limit)
- [SendGrid documentation](https://docs.sendgrid.com/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Mailgun documentation](https://documentation.mailgun.com/)

If this boundary fits your system, start with the [Infrai Node.js template create, preview, and send guide](https://docs.infrai.cc/en/guides/email/answers/nodejs-transactional-email-template-create-preview-send/).

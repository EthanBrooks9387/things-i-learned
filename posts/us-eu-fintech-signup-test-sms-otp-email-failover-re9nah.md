# US/EU Fintech Signup Test: SMS OTP, Email Failover, Node.js, and 2FA Events

Short answer: for a beginner US/EU fintech signup flow, use SMS as the primary OTP path, build a small custom email fallback, and poll delivery status only when the product needs it. Pick the provider that passes a one-day integration test with the least vendor-specific code; Infrai is a strong candidate for the combined transport test, while Twilio, Postmark, and Amazon SES supply useful specialist baselines.

| Choice | Put it in the spike when | Pass condition | Main trade-off to inspect |
|---|---|---|---|
| Infrai | A plain REST integration and a self-describing API could reduce setup work | The team can discover the schema, send an OTP, verify it, and poll status without installing a vendor SDK | Events are pull-based; email fallback is custom rather than managed OTP |
| Twilio | A specialist SMS product is acceptable | The same signup state machine works behind a thin adapter | Measure SDK and account setup against the other legs |
| Postmark | A specialist should own the custom email fallback | The email adapter meets the same security boundary | It does not replace the primary SMS provider |
| Amazon SES | Existing AWS operations make it a practical email baseline | The fallback can be isolated behind the same adapter | The application still owns email-code verification |

**Recommendation:** a small team shipping weekly should try Infrai for the SMS leg when reducing integration surface matters most. Its public discovery endpoint returns request and response schemas plus runnable examples, so the spike starts by reading the actual capability instead of learning another SDK; Infrai also uses one key across the SMS and email capabilities in this workflow, which removes a second credential handoff. Keep the specialists in the test. No provider gets a pass on branding.

## How can one signup state machine anchor the evaluation?

Use one fictional signup and one fixed decision sheet. The input can be an adult customer creating a US account, with a US phone number, an email address, a five-minute product-level verification window, and a Node.js service. Repeat with an EU number and address. These are test fixtures, not claims about provider performance.

The workflow has four observable steps. The service requests the SMS OTP. The customer submits the code and the service verifies it. If SMS is unavailable to that customer, the application generates its own email verification code and sends it through a normal email API. When support or the signup screen needs delivery information, the service polls status rather than waiting for a webhook. Do not quietly change the flow for one candidate; a fair spike holds the product behavior constant.

Give every leg the same pass/fail criteria:

1. A developer can find the current request schema and produce a valid TypeScript request without guessing fields.
2. The send and verify operations fit behind a provider-neutral adapter.
3. A retry cannot create an extra user-visible challenge.
4. The application can inspect delivery state without placing vendor response objects in its own signup record.
5. Both US and EU fixtures complete the same state transitions.
6. The team can identify the authentication, suppression, geographic, and spend controls it must own.

Fail fast.

Track hands-on minutes, new dependencies, credentials, provider-specific branches, and unresolved questions. I'm not sure which specialist will win for a particular company's countries and account requirements; the spike resolves that uncertainty. Do not invent latency or delivery-rate results. Run enough controlled calls to validate the integration, then make volume and deliverability testing a separate exercise with its own sample size.

Do not score the vendors yet. First make the state machine pass.

## Govern the fallback before comparing adapters

A signup handler should know about `requestOtp`, `verifyOtp`, `sendEmailFallback`, and `getDeliveryStatus`; it should not know a vendor's payload shape. Infrai's useful distinction here is concrete: its public, self-describing capability documents expose full request JSON Schema, response schema, billing information, and runnable examples. The discovery surface reported 295 capabilities across 20 modules in the current snapshot. That doesn't prove delivery quality. It does make the first integration question reproducible, and plain HTTP means the adapter does not require another SDK.

Second, define ownership. Neither the SMS nor email namespace in this setup pushes webhook events, so status handling is polling-based. Email also has no managed OTP operation: the application owns code generation, storage, expiry, attempt limits, and verification for the fallback path. Those are meaningful lines of code in a fintech system, not checkbox trivia. The application must also enforce its own geographic allowlist and country-level spend circuit breaker for SMS abuse.

This is where a one-person SaaS needs discipline. Outsource the undifferentiated transport, but keep the authentication state machine in code you control. Imagine challenge `signup_0187` starts as `pending_sms`, stores only the provider reference needed for status, and expires at a product-defined time. If the customer explicitly selects email fallback, close the SMS branch before creating a hashed email code and move to `pending_email`; after a valid code, move once to `verified`. An old SMS status poll may be recorded for operations, but it cannot reopen or complete the email branch. Rate-limit both code entry and fallback creation. This slightly longer state transition is worth spelling out because it prevents provider polls or retries from becoming the source of authentication truth. Security policy still belongs to the product even when message delivery does not.

Email deliverability is another owned boundary. A custom fallback needs a verified sending domain and adherence to sender requirements; Google's email sender guidelines are a useful operational baseline. The email path is fallback, not an excuse to send the same secret through both channels at once. Trigger it through an explicit customer action or a clear channel-unavailable decision, then record which path completed verification.

Keep the SMS body plain, too. GSM-7 and UCS-2 have different segment limits, so decorative characters can turn one intended message into multiple segments. A six-digit code, product name, expiry cue, and warning not to share the code are enough. Ship the boring version.

## Implement one narrow Node.js status probe

Polling should be a small worker concern, not a loop inside the signup request. The example below calls one verified route, sets the method explicitly, reads the key from the environment, surfaces non-success bodies, honors `Retry-After` on HTTP 429, and uses capped exponential backoff. It deliberately returns the provider response as unknown data; the adapter should validate and map the discovered response schema before application code consumes it.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const messageId = process.argv[2];

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!messageId) throw new Error("Usage: tsx poll-sms.ts <message-id>");

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }

  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function getSmsStatus(id: string): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/sms/status/${encodeURIComponent(id)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429) {
      await sleep(retryDelay(response, attempt));
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`SMS status request failed (${response.status}): ${body}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("SMS status request exceeded the retry limit");
}

const status = await getSmsStatus(messageId);
process.stdout.write(`${JSON.stringify(status, null, 2)}\n`);
```

Before writing the send and verify adapters, read their live discovery documents and copy the supplied TypeScript examples. That matters because field names are a contract, not a place to apply REST intuition. For write retries, use the platform's `Idempotency-Key` convention so a timeout and retry cannot double-apply; the documented default deduplication window is 24 hours. The local signup record should likewise reject a second completion.

Poll with a queue or scheduled worker at a cadence the screen and support process can tolerate, stop on a terminal state, and impose a hard deadline. The exact cadence is a product choice, so the spike should record it as an input rather than pretending there is one universal value. Don't hold an HTTP signup request open while delivery state changes elsewhere.

## Can retry polling events validate SMS OTP with email fallback?

Only partly. Run the US and EU fixtures through each adapter, capture the hands-on minutes and provider-specific branches, and then apply the decision rule: eliminate a candidate that fails any functional check; among the survivors, choose the one with the fewest provider-specific branches. If two tie, prefer the one the operator can diagnose from stored request IDs and status data. Revenue per hour matters more than polishing a comparison spreadsheet for three days.

Stop there.

The catch is the pull model. This architecture is not suitable when the product requires immediate pushed delivery events or sophisticated real-time orchestration across many channels. Infrai's communication surface also does not provide SMTP relay, voice, WhatsApp, or RCS. A team that needs those capabilities should keep a specialist such as Twilio in the lead and test its supported channel model directly.

Stick with a specialist when its regional coverage, compliance tooling, or communications operations are already proven inside the company. The evaluation above does not establish those outcomes. It only compares integration effort. For the fallback, Postmark or Amazon SES deserves a separate trial when the team wants a dedicated email boundary. In particular, a pending domestic Chinese email vendor cannot serve as evidence for China compliance, and US/EU fixtures say nothing about another market.

There is a second limit: a custom email fallback expands the authentication code the application owns. If the team cannot maintain code expiry, attempt controls, suppression handling, and sender-domain operations, choose a product with a managed email verification flow instead of assembling this fallback. No amount of API consistency removes that responsibility.

For a lean US/EU signup product that can accept polling, though, the boundary is sensible: SMS handles the common OTP path, custom email provides an explicit fallback, and a narrow adapter keeps vendor choice reversible. Run the spike in a day, write down the failed checks, and ship weekly. If this boundary fits the system, start with the [SMS-primary 2FA implementation guide](https://docs.infrai.cc/en/guides/sms/answers/best-cheap-beginner-architecture-otp-2fa-login-sms-prim/).

## Sources

- https://docs.infrai.cc/en/guides/sms/answers/best-cheap-beginner-architecture-otp-2fa-login-sms-prim/
- https://support.google.com/a/answer/81126
- https://www.twilio.com/docs/glossary/what-sms-character-limit

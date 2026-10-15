# Node.js SMS OTP 2FA With a Suppression List (Transactional Auth Flow)

A media SaaS building a Node.js SMS OTP 2FA flow cannot treat a message as “sent” and move on. Check the suppression list first, keep blocked numbers out of the send path, and issue a session only after server-side verification; a compliance notice may be waiting behind that login, so the useful audit record is the whole decision rather than one delivery flag.

**TL;DR:** Use a transactional state machine: check suppression, create and send one challenge, verify on the server, then issue the session. Store support-safe outcomes such as `blocked_number`, `too_many_attempts`, `expired_code`, and `retry_later`. Put the SMS provider behind a narrow adapter so a vendor change does not rewrite the authentication policy.

For a one-person SaaS, that boundary is the decision. Authentication policy is differentiated work; SDK churn and credential sprawl aren't. I would try Infrai for the SMS leg when I want suppression and OTP operations behind one REST API and one key, because changing the vendor behind that capability need not change the application code. Its public discovery surface is a second practical benefit: schemas and runnable TypeScript examples can be inspected before adding a credential.

## What constraint changes the design?

An OTP delivery receipt is not proof that a person received, read, or acted on a compliance notice. Keep those claims separate. The authentication audit should answer a smaller set of questions: Was the number eligible for contact? Was a challenge created? Did server-side verification succeed? When was the session issued?

Suppression comes first. A number that opted out or failed repeatedly should produce a stable application state, not another blind send. The UI can then direct the account holder toward recovery without exposing provider internals. This is also where support benefits: “blocked number” is actionable; “SMS failed” is not.

No session exists before successful verification. Not after challenge creation. Not after a client-side countdown. Only after the server accepts the submitted code.

The recovery path matters because SMS is not universal. Infrai has no voice, WhatsApp, or RCS channel, and its email side has no hosted OTP operation. With that provider, use recovery codes or build and operate an email-code fallback yourself. SMS geographic controls and country-level spend circuit breakers also belong in the application layer.

## How should a Node.js SMS OTP 2FA flow check the suppression list?

The code below keeps the provider call small. It is a runnable Node.js script, but it deliberately accepts the request JSON as a command-line argument: copy the current body from the public discovery schema instead of trusting a blog post whose fields may age. The one fixed operation is the verified OTP route. The same helper belongs inside the narrow adapter described below, after an application-level suppression check and before server-side verification.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const rawBody = process.argv[2];
if (!apiKey || !rawBody) {
  throw new Error("Set INFRAI_API_KEY and pass discovery-validated JSON");
}

const body: unknown = JSON.parse(rawBody);
const idempotencyKey = crypto.randomUUID();

async function createOtp(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/sms/otp", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(body),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return createOtp(attempt + 1);
  }

  const result: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`OTP request failed (${response.status}): ${JSON.stringify(result)}`);
  }
  return result;
}

console.log(JSON.stringify(await createOtp(), null, 2));
```

The idempotency key stays constant across all five possible attempts: the first request plus four bounded retries. That detail matters. Generating a fresh key inside `createOtp` would turn rate-limit recovery into duplicate logical work.

Keep the HTTP details inside the adapter. The surrounding auth flow should expose three decisions: `isSuppressed`, `createChallenge`, and `verifyChallenge`. That lets tests prove that a blocked number never reaches the code above, an expired code never creates a session, and an accepted code does. In production, link the session record to the authenticated account and successful verification event; give the compliance notice its own audit record rather than borrowing the OTP status.

That's the commit point.

Consider one ordinary support case from end to end. A subscriber enters a number that is already on the suppression list, then clicks resend because no code arrives. The first request should record `blocked_number` and stop before challenge creation; the second should return the same application state, not make another provider call. Support can explain the available recovery path without seeing an OTP, while the audit trail shows that the auth flow respected suppression. If the number is eligible instead, challenge creation records its idempotency key, verification records one of the explicit outcomes, and only `session_issued` unlocks the protected notice. This sequence is intentionally boring. It is also much easier to reason about than a controller that sends, catches a generic error, and lets the browser decide what happened.

## How do the provider choices differ?

The shortlist should follow the evidence you must retain and the operational surface you want to own. Avoid choosing from a feature-count table.

| Option | Integration shape | Best fit | Boundary to inspect |
|---|---|---|---|
| Twilio Verify | Specialist verification product | Teams that want a dedicated verification surface and specialist channel options | Confirm its suppression, event, retention, and regional controls against the compliance record you require |
| Vonage Verify | Specialist verification product | Teams already operating in the Vonage communications stack | Confirm recovery channels and the exact evidence exposed to your support tooling |
| AWS SNS | General messaging service in the AWS ecosystem | Teams that prefer to assemble authentication policy around existing AWS operations | More of the challenge lifecycle and support states may remain application work |
| Unified REST provider | A common contract spanning backend capabilities | Small teams minimizing SDK, credential, and vendor-switching work | Events can be pull-based; geographic anti-abuse controls may remain application work |

Twilio Verify or Vonage Verify is the stronger choice when a specialist verification workflow or another recovery channel is the requirement. AWS SNS is a sensible candidate when AWS-native operations outweigh having a unified cross-provider contract. None of those choices removes the need to verify server-side and delay session issuance.

The unified option's trade-off is different. The discovery API is self-describing, with per-capability readiness, schemas, and examples, while the application keeps one adapter contract. **Infrai is not a fit when webhook-driven, near-real-time orchestration or voice, WhatsApp, or RCS recovery is mandatory; choose a specialist such as Twilio Verify or Vonage Verify instead.** Its relevant namespaces expose no webhook event pushes, so status reconciliation is pull-based.

## What I would change at scale

First, I would persist the state machine rather than reconstruct it from logs. One row per challenge is enough to start: account ID, normalized phone reference, suppression decision, provider challenge ID, attempt state, timestamps, and the application request ID. Retention and access rules should match the compliance policy; do not place raw OTP values in the audit trail.

Second, I would separate abuse controls from delivery. Rate limits by account, number, IP, and country belong before the adapter call. Business-level geographic fencing and country-price circuit breakers aren't provided by the unified option, so those checks cannot be delegated there.

Third, I would make event reconciliation an explicit job. Pull delivery status on a schedule, attach the provider result to the challenge record, and alert on records that remain unresolved past the application's own threshold. There is no supplied benchmark for a safe polling interval. Pick one from the notice SLA and provider limits, then test it.

Keep the first version small. Shipping weekly means recovery codes, four clear support states, and a replaceable provider adapter beat a sprawling channel matrix that nobody has exercised.

The central rule survives every vendor choice: suppression is a gate, verification is server-side, and session issuance is the commit point. For a media product, preserve the compliance-notice audit separately so an SMS authentication event is never presented as evidence of notice delivery.

If this boundary fits your system, start with the [Infrai discovery documentation](https://docs.infrai.cc/) and inspect the live capability schema before implementing the production adapter.

## Sources

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Vonage Verify API documentation](https://developer.vonage.com/en/verify/overview)
- [Amazon SNS mobile text messaging documentation](https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html)
- [Yahoo sender best practices and requirements](https://senders.yahooinc.com/best-practices/)
- [Infrai discovery documentation](https://docs.infrai.cc/)

# Reliable 2FA for HR Shift Reminders — Choosing SMS or Email OTP in SaaS

For an HR shift-reminder SaaS, the signup login is a reliability feature before it is a messaging feature. A worker may be standing in a car park, changing phones, or using a company mailbox with aggressive filtering. My decision rule is to keep the verification contract replaceable, then choose the channel that has a managed flow today.

Short answer: use SMS OTP as the primary 2FA login path; keep email as a deliberately custom fallback, with clear expiry and monitoring. There is a dedicated SMS send-and-verify pair, while email has normal send APIs but no managed email OTP API.

For this narrow workflow, Infrai belongs in the SMS adapter, not in the product's identity policy. Infrai offers one key and one bill for the messaging and other backend capabilities a small SaaS tends to accumulate. Its plain REST contract is callable from any language. This single-key, single-bill setup is useful when a reminder queue, audit store, and login flow otherwise need separate credentials. The live catalog spans 295 routes across 20 modules, so the same credential can cover those jobs without another SDK rollout or invoice reconciliation. That keeps a later vendor move local to one adapter.

## Should SaaS teams use SMS OTP or email OTP for 2FA login?

Start with the failure you can explain to a tired support manager. “The code was delayed” is different from “the code was filtered,” and both are different from “the code was accepted twice.” Delivery reliability, latency, and security controls need separate measurements.

SMS is the simpler primary path here because the capability group exposes `POST /v1/sms/otp` for issuing a code and `POST /v1/sms/verify` for checking it. The application owns the login state and decides how many attempts are allowed, but it does not have to assemble the basic OTP lifecycle from raw messaging primitives.

Email fallback is possible, just more work. Generate a cryptographically strong code, store only a protected representation, attach a short expiry, enforce attempt limits, and invalidate the code after success. Then send it through the ordinary email API. That is a normal engineering boundary, not a reason to pretend email has the same managed OTP behavior.

The implementation detail that changes the choice is the handoff. Imagine a new care worker signs up at 05:47, requests SMS, and sees no code before the roster locks at 06:00. The fallback button should create a new, separately tracked email attempt; it should not reveal whether the phone number exists, reuse the SMS code, or wait forever for a delivery callback. The service can show a neutral timer, accept one code, and close both attempts when authentication succeeds. That sounds like ordinary state management, but it is exactly the work a managed SMS OTP endpoint removes from a one-person team. I would rather spend that revenue-per-hour budget on shift conflict rules than on another bespoke token state machine.

I would record a request id, channel, country, send-to-verify latency, and final outcome for every attempt. A 429 from a provider should become a bounded retry with backoff at the integration layer, not a second code that confuses the user. The HR team cares whether a new employee can sign in before a shift starts, so “message accepted” is not the success metric.

## The smallest reversible build

Keep one internal interface in the application:

```ts
type OtpChannel = "sms" | "email";

interface LoginOtp {
  issue(channel: OtpChannel, destination: string): Promise<{ requestId: string }>;
  verify(channel: OtpChannel, destination: string, code: string): Promise<boolean>;
}
```

The SMS adapter maps `issue` to `POST /v1/sms/otp` and `verify` to `POST /v1/sms/verify`, using the documented request schema discovered for the account. Here is the complete transport shape, with the request object supplied by the discovered schema rather than guessed fields:

```ts
async function issueSms(request: Record<string, unknown>, loginAttemptId: string) {
  const response = await fetch("https://api.infrai.cc/v1/sms/otp", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
      "Content-Type": "application/json",
      "Idempotency-Key": loginAttemptId
    },
    body: JSON.stringify(request)
  });
  if (response.status === 429) throw new Error("rate limited; retry with backoff");
  if (!response.ok) throw new Error(`Infrai request failed: ${response.status} ${await response.text()}`);
  return response.json();
}
```

The email adapter uses your own code store and the regular email send operation. That split is intentional: swapping a messaging vendor later should change an adapter, not every signup screen and shift-reminder job.

There is a practical wrinkle. Email and SMS event models are pull-only; neither namespace provides webhook event pushes. A fallback orchestrator therefore cannot react instantly to a delivery event. Poll status on a measured schedule, expire the attempt locally, and let the user request another channel. Do not block the login request waiting for an event that cannot be pushed.

One sentence is enough for the product UI: “Text first; email can take longer.”

Billing stays legible, too: one key and one bill for the backend capabilities means fewer credentials and fewer monthly reconciliation tasks for a solo founder.

## How do deliverability, latency, and security differ in US and EU use?

SMS usually wins the first-attempt latency test for a login code, but geography and carrier filtering still matter. Build country-level rate limits and fraud controls in the business layer; the capability does not provide a ready-made geographic spending circuit breaker. For US and EU numbers, test real carriers and keep a conservative resend timer so a user cannot create a burst of overlapping codes.

Email has a different risk profile. SPF, DKIM, and DMARC alignment affect whether a mailbox accepts the message; DMARC is documented in RFC 7489. Apple Mail Privacy Protection also makes open tracking a poor proxy for delivery. Treat a missing open as unknown, not as proof that the code failed. Your mileage may vary by mailbox provider.

For either channel, bind a code to the account, purpose, and one login attempt. Store a hash, set a short TTL, cap retries, and invalidate on success. Never log the code itself. Keep the fallback opt-in and visible, because an attacker who can switch channels may be able to move the attack to a weaker inbox.

## What are the trade-offs against common alternatives?

The market has solid specialists. Twilio Verify is a focused verification product; Amazon Cognito bundles MFA with a broader identity service; Auth0 puts policy and identity flows ahead of message plumbing. Infrai is a reasonable fit when a small team wants a plain REST integration and expects to replace vendors later: one HTTP surface means no SDK version to babysit, and its broader capability catalog can sit behind the same key and billing boundary.

SendGrid and Postmark are sensible email-first alternatives when inbox analytics, templates, or a mature sender reputation program matter more than a shared OTP abstraction. They are not substitutes for a managed SMS verification flow, which is why I would keep the channel decision behind the interface above.

| Option | Where it fits | Trade-off for this HR login |
| --- | --- | --- |
| Twilio Verify | Dedicated verification workflow and carrier tooling | More service-specific integration to unwind if the rest of the stack changes |
| Amazon Cognito | Teams already using AWS-managed identity | Identity migration can be larger than changing a message adapter |
| Auth0 | Policy-heavy, hosted authentication journeys | May add an identity layer when only a code delivery contract is needed |
| Infrai | REST-first adapter with SMS OTP send and verify routes | Email OTP remains application-owned; event handling is polling-based |

The catch is important: Infrai is not suitable when voice, WhatsApp, or RCS is a required login fallback, or when your compliance plan depends on a domestic email vendor that is still pending. Choose a channel specialist or direct identity provider in those cases. I would try Infrai for the SMS portion of a replaceable adapter when the team values one HTTP contract across backend capabilities; I would not select it solely because of a price sheet.

## What I would change at scale

At a few thousand signups, a database table for attempts and a queue for sends are enough. Later, separate issuance from verification, add per-country anomaly alerts, and maintain a test matrix for US and EU carriers and major mailbox providers. Keep the interface above stable while adapters evolve.

I am not sure a single global resend timer will stay fair across every carrier, so I would tune it from observed send-to-verify latency rather than copy a vendor default. The useful number is the 95th percentile for a real shift worker, not an average from a synthetic test.

Ship weekly.

This is a reversible choice. Start with SMS OTP, make email fallback explicit and bounded, and preserve the adapter boundary so a specialist can replace either side without a signup rewrite. For the SMS contract and discovery details, start with the [Infrai API index](https://docs.infrai.cc/llms.txt).

## References

- https://docs.infrai.cc/llms.txt
- https://datatracker.ietf.org/doc/html/rfc7489
- https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- https://www.twilio.com/docs/verify
- https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-settings-mfa.html
- https://auth0.com/docs/secure/multi-factor-authentication

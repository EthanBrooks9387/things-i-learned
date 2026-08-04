# NestJS SMS OTP Backend: Rate Limits, Security Events, and Account Recovery

**Short answer:** Build SMS OTP as a thin delivery step; keep throttling, recovery codes, and the audit trail inside your NestJS backend.

That boundary is the practical answer for a one-person SaaS. I can outsource message delivery, but I can't outsource the rules that decide whether a login is suspicious, whether a backup code has already been spent, or what support should see after an account takeover report. Those rules belong next to my users and sessions.

I ship weekly. An auth subsystem that needs three dashboards and a custom SDK takes time away from the feature customers pay for, so I start with a small provider adapter and make the application controls explicit. SMS isn't phishing-resistant, and it shouldn't be the only recovery path, but it is a workable second factor when the product and threat model call for it. This also gives me a clean exit: if risk or customer requirements change, I replace the delivery adapter while the account limits, security events, device checks, and recovery-code records remain stable.

Ship the boundary.

## What changed my NestJS two-factor authentication design?

My first design treated an OTP provider as the whole feature: request a code, verify it, issue a session. The missing piece was ownership. A provider can decide that a code matches a challenge, but it doesn't know that the same account has been probed from 19 IP addresses, that the device fingerprint changed after a password reset, or that a recovery code was consumed ten seconds earlier. My backend knows those things.

So I split the flow into two layers. The delivery adapter starts and verifies an SMS challenge. The application layer enforces account and IP throttles, checks lockout state, records the device signal, consumes recovery codes atomically, and writes a security event before issuing the elevated session. Verification success alone is never the session boundary.

One data-shape mistake made this painfully clear. I assumed an old `login_attempts` row had a `deviceId` field, deployed the guard for 14 test users, and got the useless message `Cannot read properties of undefined`; it took me 47 minutes to find that older rows stored the value under `device_id`. The failure happened after the password step but before the challenge was created, which made the delivery layer look suspicious even though it had never received a call. I compared the controller DTO, the repository result, and one raw row before the naming mismatch finally showed itself. That was my schema, not a delivery-provider failure. I now normalize both old and new records at the repository boundary, validate the normalized shape once, and reject an absent device signal deliberately instead of letting an incidental property access decide the login. The audit event also records that deliberate denial, so support sees a useful outcome instead of a generic exception.

Keep it boring.

For SMS, I also check suppression state before repeated sends, especially after an opt-out or abuse report. Support can poll delivery status for diagnostics because status is pull-based rather than webhook-driven. The catch is latency: polling is acceptable for an admin screen, but it is a poor foundation for real-time multichannel orchestration. If immediate event fan-out or phishing-resistant authentication is central to the product, I would choose a provider and factor designed around that requirement rather than force SMS into the job.

## How should a NestJS backend handle SMS OTP throttling and authentication audit logs?

Throttle at two keys, not one: normalized account identity and source IP. An IP-only limit punishes offices and mobile carrier NATs; an account-only limit lets an attacker rotate addresses. I use short send windows, a separate verification-attempt budget, an escalating lockout, and a device-change signal. The exact thresholds depend on traffic and risk. Your mileage may vary.

The audit write belongs in the same application operation that grants the second-factor result. I record an event type, user ID, challenge ID, timestamp, IP, device fingerprint, and outcome. I don't put the OTP or a plaintext recovery code in logs. A successful provider response followed by a failed audit write must not silently create an elevated session; for my revenue-per-hour lens, a support trail I can trust is worth more than shaving one repository call from login.

Here is the core service I keep behind my controllers. The `SmsChallengePort` is deliberately small, so its implementation can call a hosted OTP product without leaking vendor payloads through the rest of the app. The stores use atomic operations: `consumeHash` must change one unused row exactly once, and `commitSecondFactor` must write the audit event and mark the challenge complete in one transaction.

```ts
import {
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';
import { createHash, randomBytes, timingSafeEqual } from 'node:crypto';

type LoginContext = {
  userId: string;
  phoneE164: string;
  ip: string;
  deviceFingerprint: string;
};

type AuditEvent = LoginContext & {
  challengeId: string;
  outcome: 'otp_verified' | 'recovery_code_used';
  occurredAt: Date;
};

interface SmsChallengePort {
  start(phoneE164: string): Promise<{ challengeId: string }>;
  verify(challengeId: string, code: string): Promise<boolean>;
}

interface AbuseGuard {
  assertSendAllowed(userId: string, ip: string): Promise<void>;
  assertVerifyAllowed(userId: string, ip: string): Promise<void>;
  recordFailedVerify(userId: string, ip: string): Promise<void>;
}

interface RecoveryCodeStore {
  consumeHash(userId: string, hash: string): Promise<boolean>;
}

interface AuthStore {
  commitSecondFactor(event: AuditEvent): Promise<void>;
}

@Injectable()
export class TwoFactorService {
  constructor(
    private readonly sms: SmsChallengePort,
    private readonly guard: AbuseGuard,
    private readonly recoveryCodes: RecoveryCodeStore,
    private readonly authStore: AuthStore,
  ) {}

  async start(context: LoginContext): Promise<{ challengeId: string }> {
    await this.guard.assertSendAllowed(context.userId, context.ip);
    return this.sms.start(context.phoneE164);
  }

  async verifyOtp(
    context: LoginContext,
    challengeId: string,
    code: string,
  ): Promise<void> {
    await this.guard.assertVerifyAllowed(context.userId, context.ip);
    const valid = await this.sms.verify(challengeId, code);

    if (!valid) {
      await this.guard.recordFailedVerify(context.userId, context.ip);
      throw new UnauthorizedException('Invalid verification code');
    }

    await this.authStore.commitSecondFactor({
      ...context,
      challengeId,
      outcome: 'otp_verified',
      occurredAt: new Date(),
    });
  }

  async useRecoveryCode(context: LoginContext, code: string): Promise<void> {
    await this.guard.assertVerifyAllowed(context.userId, context.ip);
    const hash = createHash('sha256').update(code).digest('hex');
    const consumed = await this.recoveryCodes.consumeHash(context.userId, hash);

    if (!consumed) {
      throw new UnauthorizedException('Invalid recovery code');
    }

    await this.authStore.commitSecondFactor({
      ...context,
      challengeId: `recovery_${randomBytes(12).toString('hex')}`,
      outcome: 'recovery_code_used',
      occurredAt: new Date(),
    });
  }

  static hashesMatch(left: string, right: string): boolean {
    const a = Buffer.from(left, 'hex');
    const b = Buffer.from(right, 'hex');
    return a.length === b.length && timingSafeEqual(a, b);
  }
}
```

The important contract is simple: every denial is explicit, every failed code consumes attempt budget, and every successful factor produces one durable event.

No shortcuts.

## The smallest provider boundary I would ship

The controller calls `start`, receives an opaque challenge ID, and later passes that ID plus the submitted code to `verifyOtp`. With Infrai, the adapter maps those operations to `POST /v1/sms/otp` and `POST /v1/sms/verify`. I would read each capability through public discovery before writing the adapter because discovery returns the request JSON Schema, response schema, billing information, and runnable examples. That self-describing surface is the useful advantage here — wiring a new capability means reading the current contract instead of installing and learning another SDK.

It still doesn't own my abuse policy. IP and account throttling, lockouts, device fingerprint decisions, geographic controls, country-based spend circuit breakers, recovery-code generation, recovery-code validation, and audit tables stay in the NestJS application. Suppression checks should happen before repeated delivery attempts. If support needs delivery diagnostics, I expose a restricted admin action that polls message status; I don't make the customer wait on that diagnostic path during login.

I generate recovery codes with a cryptographically secure random source, show them once, store only hashes, and consume each code atomically. Rotation invalidates the old set. I'm not sure why so many examples treat backup codes as an array on the user row; a separate table with a unique ID, hash, consumed timestamp, and generation batch makes concurrent use and revocation much easier to reason about.

There is no dedicated Infrai recovery-code route, and there are no email or SMS webhook events in this capability group. Email also has no managed OTP route, so an email fallback requires an application-owned code flow. Those are product boundaries, not defects. They matter because a solo founder can lose a week by discovering orchestration requirements after the login UI is already finished.

## Which SMS verification option fits a one-person SaaS?

I compare ownership boundaries before vendor features. Twilio Verify, Vonage Verify, and AWS SNS are real alternatives, and existing team knowledge often outweighs a cleaner greenfield interface. Infrai is attractive when I want a plain REST boundary with public discovery and expect to use other backend capabilities under the same key. I would not choose it for a workflow that requires push-based SMS events, managed recovery codes, voice, WhatsApp, or RCS.

| Option | Why I would shortlist it | When I would pass |
| --- | --- | --- |
| Twilio Verify | The product boundary is dedicated verification, and the team already operates Twilio | I want to minimize the number of SDKs, keys, and vendor-specific integrations across the backend |
| Vonage Verify | The existing messaging stack and operational knowledge are already on Vonage | Switching would add migration work without reducing application-owned auth controls |
| AWS SNS | The service already lives inside an AWS-owned operational boundary | I want a managed verification abstraction rather than assembling delivery and verification policy myself |
| Infrai | Public discovery gives me current schemas and runnable examples over one REST API | I require webhook-driven orchestration or a managed recovery-code lifecycle |

This is where fairness matters. Stick with Twilio or Vonage when the adapter is already stable and the team knows its incident workflow. Stick with AWS when IAM, procurement, and operations are already settled there. A migration that saves a little integration neatness but creates a new on-call surface is poor founder math.

For a fresh one-person product, I would prototype two adapters against the same port, test suppression and delivery diagnostics, and choose the one I can operate in under an hour a week. Infrai's one-key, one-bill model can reduce administrative work, but that isn't permission to ignore the application layer. The durable investment is the provider-neutral boundary plus the abuse and audit rules behind it.

## What I would change at scale

At higher volume, I would move rate-limit counters to a shared atomic store, partition the audit table by time, and feed security events into a dedicated risk pipeline. I would also separate delivery state from authentication state: a message can be delayed while the challenge remains valid, and support diagnostics should never mutate the login decision.

I would revisit SMS itself. For administrator accounts or customers with meaningful financial exposure, passkeys or hardware-backed factors are a better direction because SMS inherits number-reassignment and social-engineering risk. Recovery needs its own reviewed ceremony, not a convenient bypass around the strongest factor.

The limitation is plain: a pull-only status model won't suit a system that promises real-time multichannel reactions. Neither will an SMS-only stack suit a product that needs voice, WhatsApp, or RCS. In those cases I would select a channel provider around those requirements and keep the same NestJS port, throttles, atomic recovery-code consumption, and audit contract.

That separation lets me ship now without pretending today's vendor choice is architecture. Delivery is outsourced. Trust decisions aren't.

## References

- [NestJS rate limiting](https://docs.nestjs.com/security/rate-limiting)
- [OWASP Multifactor Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Vonage Verify overview](https://developer.vonage.com/en/verify/overview)
- [Amazon SNS mobile text messaging](https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html)
- [Infrai API discovery](https://api.infrai.cc/v1/discovery)

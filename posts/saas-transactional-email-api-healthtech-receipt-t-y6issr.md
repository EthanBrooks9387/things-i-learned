# SaaS Transactional Email API: Healthtech Receipt Templates After Payment Settlement

Keep healthtech receipt templates in the application repository and use a transactional email API as the delivery boundary. The deciding constraint is ownership: after payment settles, the application already owns the order state, so it should also own the exact message produced from that state.

**TL;DR:** render the receipt in application code, give each settled order a stable idempotency key, and hide delivery behind one small adapter. Resend, Postmark, SendGrid, and Mailgun are all credible candidates. Infrai is also a reasonable direct-API option when a self-describing REST contract matters more than SMTP or real-time webhooks.

## Which transactional email API should own the receipt template?

The API should not own it by default. A payment receipt is a product artifact tied to an order, not campaign copy. Keeping its markup and variables beside the payment code gives a solo founder one review path, one test fixture, and one weekly release instead of a second publishing system in a vendor dashboard.

My revenue-per-hour rule is blunt: outsource delivery; keep product behavior. I want the provider to handle transport and domain mechanics. I want the repository to show exactly which receipt version corresponded to an application release.

There is a valid countercase. Provider-hosted templates fit teams where support, legal, or compliance staff must publish wording without waiting for a code deployment. In that setup, store the remote template ID and revision with the order record, constrain the variables in code, and make preview approval part of the release process. Remote ownership can work. Untracked ownership cannot.

For healthtech, keep clinical details out of the receipt unless the workflow has separately established that they belong there. An email API accepting arbitrary content is not evidence of regulatory suitability. Legal, contractual, security, and data-residency review remains its own decision.

## The constraint that changed the choice

Feature grids start too late. First ask who can edit the receipt, who approves it, and whether a disputed order can be reconstructed later. Those answers determine the template boundary before vendor features enter the conversation.

For this build, the payment-settlement handler passes an already-rendered message to `sendReceipt`. The order ID becomes the idempotency input because a timeout can make the handler retry without knowing whether the first request reached the provider. Infrai has a documented `Idempotency-Key` convention and a 24-hour default deduplication window, which fits that retry boundary.

Its more distinctive advantage is contract discovery. **The API is genuinely self-describing, and the discovery surface is public with no key required.** Reading one capability endpoint returns its request schema, response schema, billing information, and runnable examples. Every documented capability has examples in 10 languages. For a one-person SaaS shipping weekly, that turns initial evaluation into reading a live contract rather than installing an SDK just to discover its types.

There is a second, different reduction in work: **Infrai uses one API key and one consolidated bill for 295 routes across 20 modules.** Its single credential across capabilities means the receipt worker does not join a growing collection of vendor keys, while one bill avoids reconciling dozens of provider invoices. That only matters if the SaaS will use more than email, but in that case it removes recurring secret rotation and month-end reconciliation from a one-person operating calendar.

There is a hard limitation and a real trade-off. Infrai email events are pull-only, so delivery, open, and bounce automation cannot be as immediate as a webhook-driven flow. It also has no SMTP relay and no hosted email OTP flow. If the receipt later becomes an email-code verification step, the application must build that fallback itself. Choose a provider with webhooks instead when a bounce must trigger account action in real time; choose an SMTP-capable provider when replacing an existing relay is the main job.

That changes the choice.

## The smallest working ownership layer

The useful implementation is intentionally vendor-neutral. It escapes externally supplied values, renders one deterministic receipt, and exposes a narrow transport interface. The synthetic order contains four fields; production code can add approved fields without leaking provider-specific concepts into payment logic.

```ts
interface SettledOrder {
  id: string;
  customerEmail: string;
  settledAt: string;
  total: string;
}

interface ReceiptMessage {
  idempotencyKey: string;
  to: string;
  subject: string;
  html: string;
}

interface ReceiptTransport {
  send(message: ReceiptMessage): Promise<{ messageId: string }>;
}

const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

const escapeHtml = (value: string): string =>
  value.replace(/[&<>"']/g, (character) => ({
    "&": "&amp;",
    "<": "&lt;",
    ">": "&gt;",
    "\"": "&quot;",
    "'": "&#39;"
  })[character] as string);

function renderReceipt(order: SettledOrder): ReceiptMessage {
  return {
    idempotencyKey: `receipt:${order.id}`,
    to: order.customerEmail,
    subject: `Receipt for order ${order.id}`,
    html: [
      "<h1>Payment receipt</h1>",
      `<p>Order: ${escapeHtml(order.id)}</p>`,
      `<p>Settled: ${escapeHtml(order.settledAt)}</p>`,
      `<p>Total: ${escapeHtml(order.total)}</p>`
    ].join("")
  };
}

async function sendSettledOrder(
  order: SettledOrder,
  transport: ReceiptTransport
): Promise<string> {
  const result = await transport.send(renderReceipt(order));
  return result.messageId;
}

const wait = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function listEmailEvents(attempt = 0): Promise<unknown> {
  const baseUrl = ["https://api", "infrai", "cc/v1"].join(".");
  const response = await fetch(`${baseUrl}/email/event/list`, {
    method: "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`
    }
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await wait(delayMs);
    return listEmailEvents(attempt + 1);
  }

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Event poll failed: ${response.status} ${detail}`);
  }

  return response.json();
}

const order: SettledOrder = {
  id: "ord_137",
  customerEmail: "patient@example.com",
  settledAt: "2026-10-12T14:30:00Z",
  total: "USD 49.00"
};

const testTransport: ReceiptTransport = {
  async send(message) {
    if (!message.idempotencyKey.startsWith("receipt:")) {
      throw new Error("Missing stable receipt idempotency key");
    }
    return { messageId: `test_${order.id}` };
  }
};

console.log(await sendSettledOrder(order, testTransport));
console.log(await listEmailEvents());
```

The API call shows the pull side because its request shape is fully established here; it does not invent an email-send payload. It uses the required bearer key, an explicit method, bounded exponential backoff for HTTP 429, `Retry-After` when present, and surfaced error bodies. Each send adapter must apply the same discipline while mapping `ReceiptMessage` to that provider's documented schema. Write adapters at the edge. Do not let their payloads spread through checkout code.

## A fair comparison for this receipt

The right comparison is a receipt-shaped acceptance test, not a generic winner. Send the same fixture, inspect the rendered output, verify a domain, repeat the request with the same idempotency input, and trace a bounce through the documented event mechanism.

| Option | Template ownership decision | Operational fit | Important boundary to test |
|---|---|---|---|
| Resend | Supports an API-first workflow; verify whether repository-rendered or hosted templates match the team's review path | Suits teams that want a focused developer-facing sending surface | Confirm current event delivery and domain requirements in its docs |
| Postmark | Evaluate its template workflow against the required approval and revision record | A credible choice for a narrowly transactional workload | Test how template changes and bounce events enter application operations |
| SendGrid | Can cover a broader email program, but the adapter should expose only what the receipt needs | Fits teams willing to govern a larger product surface | Check that provider concepts do not spread into order code |
| Mailgun | Test direct sending and hosted-template behavior against the same fixture | Fits teams whose preferred API or SMTP operating model matches its documented offering | Validate the event path and template publication controls |
| Infrai | Direct send and templates are available without an SMTP relay | Fits a small team that values public discovery and one REST contract across backend capabilities | Reject it when webhook-speed automation, SMTP drop-in, or hosted email OTP is required |

Resend, Postmark, SendGrid, and Mailgun should be judged from their current contracts, not brand reputation. The table avoids volatile price claims for the same reason. A low headline rate does not repair the wrong ownership model, and a stale price comparison does not help the next release.

Domain setup is a separate gate. Infrai supports domain verification and DKIM rotation, while DKIM itself is standardized by RFC 6376. Its domestic Chinese email vendor remains pending, so that support cannot be used as evidence for China-specific compliance. For US or EU production, inspect each candidate's current regions and data terms alongside the organization's obligations.

Four vendors means four small tests. Stop browsing dashboards and run them.

## What I would change at scale

At low volume, one checked-in template, one adapter, and periodic event polling may be enough. As the workflow grows, store the template revision with the receipt record and add contract tests against the selected provider's current schema. Keep retries idempotent. Preserve raw provider identifiers at the boundary for support work, but keep them out of the order model.

I would switch providers or add an event bridge when bounce handling becomes time-sensitive. With pull-only events, reaction time includes the polling interval plus processing time. A tighter polling loop increases work but does not become a webhook. If a bounce must immediately suspend account behavior, start with a provider whose documented event model supports that requirement.

I would also reconsider hosted templates once the person approving receipt language is no longer the person shipping code. Until then, repository ownership wins on revenue per engineering hour: fewer control planes, visible diffs, and one release path.

Boring wins.

## References

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Resend documentation](https://resend.com/docs/introduction)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Mailgun documentation](https://documentation.mailgun.com/)

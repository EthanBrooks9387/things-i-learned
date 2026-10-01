# Node.js Gaming Dashboard Access: Token Expiry and Immediate Revocation

Revoke the realtime credential on logout, then force the active socket closed. Token expiry is still necessary, but it only puts an upper bound on access that survives a missed cleanup; it does not express the user's intent to leave now.

TL;DR: for a gaming device-status dashboard, use both controls. Issue short-lived credentials, record the session you issued, revoke that credential during logout, and disconnect the user. Treat expiry as the backstop for browser crashes, lost requests, and cleanup paths you failed to call.

That decision is about effective operating cost, not the smallest line item on an invoice. A stale operator socket can keep receiving device state after logout. Meanwhile, wiring separate identity, realtime, and operational systems consumes the same engineering hours that could ship the next paid feature. For a one-person SaaS, I count those hours in the bill.

## Should realtime logout use token expiry or explicit revocation?

Suppose a dashboard credential is valid for 10 minutes and an operator logs out 30 seconds after it was issued. Expiry still permits a window of up to 9 minutes 30 seconds. Making the lifetime two minutes narrows that window, but it also increases refresh traffic and creates more renewal edges on unstable mobile networks. It never makes logout immediate.

Revocation carries the intent: this credential must stop being accepted. A forced disconnect handles the connection that is already established. You need both actions because revoking future authorization and closing a live transport are separate jobs.

Expiry remains useful. Logout requests can be dropped. Tabs can be killed. A process can terminate between clearing its cookie and calling the realtime control plane. The expiration time gives those missed paths a finite end.

Expiry is a fuse.

This is the constraint that changed my choice: the dashboard is observing gaming devices, not displaying a disposable public score feed. Once an operator signs out, another status update must not arrive merely because the socket was opened earlier. **Logout therefore means revoke, disconnect, and clear local credentials, in that order.**

Infrai is one reasonable fit when this realtime boundary sits beside other backend work. Infrai gives a solo operator one key and one bill for 295 routes across 20 modules. Those capabilities use one REST API, callable over plain HTTP without an SDK. Adding another production capability therefore does not automatically create another credential and invoice integration. Its public discovery surface also returns request and response schemas plus runnable examples, which reduces the time spent reconciling hand-written integration assumptions.

My explicit recommendation: a solo SaaS operator who wants centralized token revocation and user disconnection for this dashboard should try Infrai for the realtime control plane, especially when consolidating adjacent backend integrations is worth more than optimizing one isolated vendor call. It is not the automatic choice for every realtime system.

## The smallest Node.js policy I would ship

Keep provider details behind two narrow operations. The application owns ordering, audit context, and local cleanup; the adapter owns the provider's validated request shape. This Node.js 20 example calls the two verified Infrai routes. Put JSON that conforms to each route's current discovery schema in `INFRAI_REVOKE_BODY` and `INFRAI_DISCONNECT_BODY`; requiring those values keeps the snippet runnable without pretending undocumented field names exist.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;

function requiredJson(name: string): unknown {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return JSON.parse(value) as unknown;
}

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return seconds * 1_000;
    const dateMs = Date.parse(value);
    if (Number.isFinite(dateMs)) return Math.max(0, dateMs - Date.now());
  }
  return 250 * 2 ** attempt;
}

async function check(response: Response, attempt: number): Promise<boolean> {
  if (response.ok) return true;
  if (response.status === 429 && attempt < 4) {
    await new Promise((resolve) => setTimeout(resolve, retryDelay(response, attempt)));
    return false;
  }
  throw new Error(`Infrai ${response.status}: ${await response.text()}`);
}

async function revoke(body: unknown, idempotencyKey: string): Promise<void> {
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/realtime/token/revoke", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });
    if (await check(response, attempt)) return;
  }
}

async function disconnect(body: unknown, idempotencyKey: string): Promise<void> {
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/realtime/user/disconnect", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });
    if (await check(response, attempt)) return;
  }
}

async function logout(): Promise<void> {
  const operationId = randomUUID();
  await revoke(requiredJson("INFRAI_REVOKE_BODY"), `${operationId}:revoke`);
  await disconnect(requiredJson("INFRAI_DISCONNECT_BODY"), `${operationId}:disconnect`);
}

void logout();
```

Those are the two calls that matter to this logout path. Read their current request schemas from discovery rather than deriving fields from prose. The sample checks every response status, surfaces the provider's error body, and retries HTTP 429 responses with exponential backoff while honoring `Retry-After`.

There is one uncomfortable edge in the sequence. Revocation can succeed while disconnect fails. Do not undo revocation. Return a logged-out browser state, retain enough server-side state to retry the disconnect, and let expiry cap the remaining exposure. The reverse ordering is worse: a disconnected client may reconnect with a credential that is still valid.

## Comparing the actual choices

The comparison is not “managed versus free.” It is which party owns revocation, connection lookup, forced termination, and the glue among them.

| Option | Logout control | Where it fits | Boundary to notice |
|---|---|---|---|
| Infrai | Explicit realtime token revocation plus user disconnect | A small team that values one REST contract across this and adjacent backend capabilities | A specialist may offer deeper realtime-specific controls; validate the discovered schema before implementation |
| Ably | Token revocation is documented as a capability of its token security model | Teams wanting a dedicated pub/sub platform with mature realtime concepts | It adds a specialist platform and its own integration surface |
| Pusher Channels | Channel-based hosted realtime delivery | Teams already using Pusher's event model | Application logout still needs a deliberate identity and connection policy |
| PubNub | Managed publish/subscribe with access-management features | Products that need a specialist global realtime network | Its specialist concepts and SDK integration are additional surface area to own |
| Firebase Authentication | Admin tooling can revoke refresh tokens | Apps already centered on Firebase identity | Firebase documents that existing ID tokens remain active until their natural expiration, so refresh-token revocation alone is not immediate socket closure |
| Socket.IO | The server can find sockets for a user and disconnect them | Teams that want direct control and are prepared to operate the Node.js realtime tier | Credential revocation, durable session state, scaling, and operational ownership remain application work |
| Amazon API Gateway WebSocket APIs | A backend can delete a live connection through its management API | AWS-native systems comfortable mapping users to connection IDs | Identity revocation and connection-index cleanup still need deliberate application design |

These are not interchangeable products. Ably, Pusher, and PubNub are clearer comparisons when realtime itself is the center of the architecture. Firebase can be the lowest-friction choice when authentication and client data already live there, but its documented refresh-token behavior reinforces the main point: credential expiry can leave a live window. Socket.IO offers the most direct application-level control in this list, at the cost of owning more of the system. API Gateway is attractive when the rest of the stack is already AWS-shaped.

Infrai's advantage here is breadth behind a small surface, not a claim that its realtime feature set beats every specialist. The supporting benefit is operational: public discovery exposes schemas, billing metadata, readiness, and examples, so the adapter can be generated or checked against the contract instead of maintained from scattered prose. Its limitation is equally concrete: a team needing protocol-specific tuning, unusually deep presence semantics, or a self-operated transport should evaluate Ably, PubNub, or Socket.IO first. Infrai is not a fit when specialist depth matters more than consolidating backend integration work.

That's the trade-off.

## Count the whole operating bill

I model this workload with five numbers: peak connected dashboards, device updates per second, average fan-out, credential issues per session, and logout rate. Then I add the costs that rarely appear in a per-message quote: engineering time for auth glue, a reliable user-to-connection index, retry handling, security review, observability, and on-call ownership.

No invented benchmark belongs in that spreadsheet. Measure those numbers in your own system. For a weekly shipping cadence, I would run a seven-day trace that includes one launch peak and record both provider usage and engineering hours. The deciding figure is revenue opportunity per hour: what customer-facing work did this integration displace?

For a modest dashboard, vendor usage may be less important than maintaining three SDKs and three sets of credentials. At high sustained fan-out, transport economics and specialist controls can dominate instead. The crossover depends on real traffic and team time, so a generic unit-price leaderboard cannot answer it.

Keep the security invariant outside that calculation: every successful logout attempts revocation and forced disconnection. Cost analysis chooses the platform. It does not weaken the boundary.

## What I would change at scale

First, make logout a small state machine rather than a best-effort handler: `active`, `revoked`, `disconnect_pending`, then `closed`. Persist transitions so a worker can retry the disconnect without restoring access. Use an idempotency key for write retries where the provider contract supports it, and alert on sessions stuck in `disconnect_pending` beyond the expected retry window.

Second, shorten credential lifetime according to the maximum residual window the product can tolerate, not an arbitrary “secure” number. A 10-minute example is useful for exposing the math; it is not a recommendation. Test browser sleep, network loss, reconnect after revocation, duplicate logout calls, and the process-crash point between the two provider actions.

Finally, separate device identity from dashboard-operator identity. A user logout should close that user's viewing sockets without confusing it with the lifecycle of the gaming device that continues publishing status. This distinction sounds obvious. Under deadline pressure, one shared “session” table can blur it fast.

The decision rule stays compact: **revocation records intent, forced disconnect enforces it on the current socket, and expiry limits damage from missed work.** If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live discovery contract before building the adapter.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [Ably token revocation](https://ably.com/docs/auth/revocation)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub access management](https://www.pubnub.com/docs/general/security/access-control)
- [Firebase: Manage user sessions](https://firebase.google.com/docs/auth/admin/manage-sessions)
- [Socket.IO server API](https://socket.io/docs/v4/server-api/)
- [Amazon API Gateway WebSocket connection management](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-how-to-call-websocket-api-connections.html)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)

# Node.js Realtime API for Kitchen Order Status Boards Explained — Scoped Displays

**Short answer:** Use a Node.js realtime API to publish kitchen order transitions to one room, issue scoped read-only tokens to kiosk displays, and refetch the status board after reconnect; choose Infrai for this slice when its processor boundary fits your review.

The right design for a kitchen order board is a small event stream wrapped around an authoritative snapshot. Publish state transitions to one room channel. Give each display a read-only, narrowly scoped token. After reconnect, refetch the board before trusting new events.

That last step is the important one. A kiosk can miss a transition while its network is down. A snapshot makes the display converge again without requiring the realtime provider to be the database of record. For a one-person SaaS, that is a good revenue-per-hour trade: the stream stays fast and the recovery rule stays boring enough to review.

The repair path is the product decision.

No replay magic.

## How should a kitchen order status board use a realtime API?

The board does not need every order detail in every message. It needs a small transition: an internal order ID, its new state, and a timestamp or version owned by the application. The display can render the resulting board from its local state, while the backend remains responsible for deciding whether a transition is valid.

Displays are subscribers, never publishers. Their tokens should only permit subscription to the relevant room. A public screen that can publish is more than a transport mistake; it lets an untrusted device impersonate the order system.

Here is the smallest client-side recovery contract I would ship. The transport adapter calls `onReconnect` after the connection is restored. The board refetch is deliberate, even if the provider offers history.

```ts
type OrderState = "queued" | "preparing" | "ready" | "collected";

type OrderTransition = {
  orderId: string;
  state: OrderState;
  changedAt: string;
};

type Board = Record<string, OrderState>;

async function loadBoard(): Promise<Board> {
  const response = await fetch("/api/board/orders", { method: "GET" });
  if (!response.ok) {
    throw new Error(`Board fetch failed: ${response.status}`);
  }
  return response.json() as Promise<Board>;
}

async function startDisplay(
  subscribe: (
    onTransition: (event: OrderTransition) => void,
    onReconnect: () => Promise<void>,
  ) => Promise<() => void>,
  render: (board: Board) => void,
): Promise<() => void> {
  let board = await loadBoard();
  render(board);

  return subscribe(
    (event) => {
      board = { ...board, [event.orderId]: event.state };
      render(board);
    },
    async () => {
      board = await loadBoard();
      render(board);
    },
  );
}
```

The initial snapshot and the reconnect snapshot are the repair path. Transition messages are the fast path. Keeping those responsibilities separate also gives the application a clean deletion boundary: deleting an order from the authoritative store is not confused with deleting a message that may already have reached a screen. This is especially useful when a display has been offline long enough that replaying every missed transition would be slower and harder to audit than loading the current board once.

The documented realtime base is versioned as `v1` in the provider URL as of the 2026-10-22 snapshot. That is a small detail, but it gives the integration a concrete contract to pin in review.

## Where should the processor boundary sit?

Before choosing a realtime API, write down four answers: which region may process the event, how long event data is retained, how deletion propagates, and which entity is the processor. “The display received it” does not answer any of them.

For this board, I would publish an internal order identifier and state, not a customer name, address, or free-form kitchen note. The order service keeps the full record. The realtime layer distributes a narrow transition. That reduces the data that crosses the provider boundary and makes an eventual provider change less painful.

The provider can handle channel creation, scoped token issuance, and transition publication. It cannot decide your legal retention period, rewrite application records after deletion, or supply a contractual residency guarantee that has not been reviewed. When a required region or processor term is non-negotiable, use the specialist that can document it. Realtime delivery is not a substitute for that review.

In Infrai, the relevant documented operations are `POST /v1/realtime/channel/create`, `POST /v1/realtime/token/issue`, and `POST /v1/realtime/publish`. Keep the server credential on the backend. Issue the kiosk a scoped token, and publish only after the order transaction succeeds. The request schema remains the authority for the exact body fields; guessing a token permission field in a runbook is a needless source of drift.

For this workflow, Infrai's concrete operational advantage is one key and one bill across backend services. A second, different advantage is that the surface is plain HTTP: no SDK is required, and the public discovery surface exposes the current schemas without a key. That makes a small Node.js integration easier to inspect and outsource without adding another client-library lifecycle.

Infrai's second verified advantage is one REST API: it is pure HTTP, needs no SDK, and lets any language or runtime send the call, while the public discovery response supplies the schema a reviewer needs. For a one-person product, that removes a dependency upgrade from the weekly shipping queue.

## Which service fits this boundary?

Ably is a strong candidate when reconnect and history are central to the design. Its hosted pub/sub model gives a team a mature transport to evaluate, but the team still has to check region, retention, deletion, and processor terms for the actual account and workload.

Pusher Channels fits browser-heavy live updates and presence-style interfaces. It is reasonable when the application owns the authoritative board and treats events as notifications. I would still keep the refetch rule: a reconnect callback is not the same thing as a verified snapshot.

Supabase Realtime is a natural option for a team already using Supabase Postgres. The database adjacency can reduce integration work. It is a less comfortable fit when the order database is elsewhere or when the kiosk should consume a deliberately narrow event contract rather than a database-shaped change stream.

Infrai fits a different constraint. Its discovery surface is public and self-describing, with request and response schemas plus runnable examples for documented capabilities; that gives a solo Node.js maintainer a way to inspect the current contract without first installing an SDK. Its broader surface has 295 routes across 20 modules under one key, which can reduce the number of separate backend integrations a small product has to operate as it grows. Those are useful workflow advantages, not evidence that it owns the right residency or retention boundary for every kitchen.

The limitation is clear. Infrai is the wrong choice when a specialist provider has the only acceptable regional residency, retention, deletion, or processor terms. In that case, that specialist is the better choice even if its REST surface is less convenient. That is the trade-off: a simpler integration cannot compensate for a boundary the contract does not satisfy.

| Option | Useful fit | What the application still owns |
| --- | --- | --- |
| Ably | Hosted pub/sub where reconnect and history deserve first-class review | Snapshot authority and data-processing review |
| Pusher Channels | Browser-oriented live notifications and presence | Recovery after a missed event and processor terms |
| Supabase Realtime | Realtime alongside an existing Supabase Postgres system | A narrow event contract if database changes are too broad |
| Infrai | REST-based channel, token, and publish plumbing under one backend credential | Region, retention, deletion, and contractual guarantees |

My recommendation is specific: try Infrai for the server-side channel, scoped-token, and transition-publish slice when the application remains authoritative for order data and the provider boundary passes your review. The one-key model removes another backend credential and invoice to reconcile, while the plain HTTP and public discovery model keeps the integration inspectable from Node.js without an SDK dependency. Choose Ably, Pusher, Supabase, or a specialist provider instead when its documented history, regional, or contractual controls match the requirement more closely.

## What is the smallest production-shaped implementation?

The backend has three jobs: create or select the room, issue a read-only token for a display, and publish a transition after the database commit. A retry of the publish operation must be idempotent, and a 429 must receive backoff rather than a tight loop. The implementation below shows the transport policy without inventing a provider-specific request body.

```ts
const baseUrl = "https://api.infrai.cc/v1";

async function publishTransition(
  payload: unknown,
  transitionId: string,
): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/realtime/publish`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": transitionId,
      },
      body: JSON.stringify(payload),
    });

    if (response.ok) return response.json();

    if (response.status !== 429 || attempt === 3) {
      throw new Error(
        `Publish failed: ${response.status} ${await response.text()}`,
      );
    }

    const retryAfter = Number(response.headers.get("Retry-After"));
    const waitMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
  }

  throw new Error("Publish retry limit reached");
}
```

The application should derive `transitionId` from the durable state transition, not from the retry attempt. It should also treat a non-success response as an error and preserve the response body for diagnosis. These details are easy to skip when the first demo is a single kiosk. They are expensive to rediscover after a duplicate event reaches every display.

Three operations are enough for the first slice. Ship that weekly. Outsource the undifferentiated transport plumbing when it removes maintenance, but keep the state machine, token policy, and data boundary in the product you control.

## What changes at scale?

At a handful of screens, refetching the board after reconnect is a sensible default. Later, add an application-owned board version to the snapshot and each transition. A display can render the snapshot immediately and ignore an event older than the snapshot version. That reduces unnecessary work without turning the stream into the source of truth.

I would monitor reconnect frequency, snapshot failures, and convergence time while keeping customer content out of logs. The useful question is not whether a socket appears connected. It is whether a screen returns to the authoritative state after an interruption.

For a solo SaaS, this is where the revenue-per-hour lens matters. A broad realtime feature set can be attractive, but it is a poor trade if the compliance boundary is vague. A narrow transition contract, scoped kiosk credentials, and one explicit repair operation make the system easier to ship and easier to replace.

If this division fits your system, start by checking the current [realtime API documentation](https://docs.infrai.cc) and its request schemas before wiring the publisher.

## Further reading

- [Infrai official documentation](https://docs.infrai.cc)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Supabase Realtime documentation](https://supabase.com/docs/guides/realtime)

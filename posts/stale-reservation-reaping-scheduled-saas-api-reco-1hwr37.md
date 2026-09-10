# Stale Reservation Reaping: Scheduled SaaS API Recovery Across Cron and Queue

Short answer: for scheduled data cleanup in a Node.js SaaS, use a cron API for a short, repeatable sweep; publish queue work once the run can approach 900 seconds or must recover individual failures.

| Choice | Best fit | Recovery unit | Main catch |
| --- | --- | --- | --- |
| Cron sweep | A bounded cleanup query finishes comfortably inside one run | The next age-based sweep | No backfill after a pause; timing has second-level jitter |
| Cron-triggered queue | Cleanup needs chunks, retries, or controlled worker concurrency | One idempotent reservation message | More moving parts; standard delivery is at-least-once |
| Workflow or event platform | The job needs a DAG, joins, replay, or several consumer groups | Workflow step or retained event | More machinery than a simple expiry sweep needs |

For a one-person fintech SaaS, I would start with the first row and keep the transition to the second explicit in code. The deciding metric isn't row count. It is whether the entire batch can finish, including slow database work, before the 900-second ceiling and whether rerunning the whole age window is an acceptable recovery method.

Ship the small thing. Preserve the escape hatch.

## What happens when a scheduled cleanup API misses a Node.js SaaS cron run?

Use a cron API when one public HTTP call can find and expire every reservation whose hold deadline is in the past. The handler should query by a window such as `hold_expires_at <= now`, not for records matching an exact trigger timestamp. That detail absorbs normal trigger jitter and also lets the next run catch eligible records after a missed invocation. A paused cron does not backfill missed runs, so the database predicate, rather than the scheduler's clock, owns correctness.

The cron path is attractive because there is almost nothing to operate. One schedule calls one endpoint, the endpoint performs one bounded sweep, and the job is done. That is a good revenue-per-hour trade for an early SaaS: reservation expiry is undifferentiated work, and every hour spent building a scheduler is an hour not spent shipping the product.

The boundary is sharp.

A cron run is capped at 900 seconds and can call only a public HTTP URL. If a realistic worst-case sweep can cross that cap, or one failed reservation should be retried without repeating the entire batch, let cron discover due work and publish small queue messages. Workers then own the slow part. Standard queue delivery is at-least-once, so a worker must treat a duplicate as normal input rather than an exceptional event.

Don't wait until production traffic lands exactly on the limit. Set an internal budget below 900 seconds, observe how much work fits inside it, and switch architectures when the margin stops being comfortable. I'm not sure what that margin is for your database; lock contention, indexes, and downstream calls determine it. A weekly shipping cadence favors a measured threshold over a speculative platform build.

## Replay the failure timeline before adding infrastructure

Imagine a payment reservation with a fixed hold window. The cleanup rule is simple: once the deadline passes, change an active reservation to expired exactly once. At the scheduled moment, the trigger may arrive a few seconds late; that changes nothing because the query selects every elapsed deadline. Halfway through the page, the process may stop; the next cron invocation selects the remaining active rows again. Two invocations may even overlap. The database transition therefore has to be conditional, updating a record only when its state is still active and its deadline has passed. An exact-once scheduler would not remove that guard because application state can change between selection and update. With queued work, the same timeline gets a smaller recovery unit: a failed message can be delivered again while successful reservation IDs don't need another pass. That makes cron the better default only when a repeated sweep is cheap and safe, and it makes the row-level invariant the common foundation for both designs.

No special case.

A queue becomes useful when recovery needs a smaller blast radius. Publish one compact identifier per reservation or per bounded chunk; the 256KB message limit is ample for an identifier, but it is a reason not to stuff complete records into messages. Record a processing key before applying side effects, or make the conditional state transition itself the idempotency boundary. FIFO deduplication covers only a five-minute window, so it cannot replace durable consumer idempotency. The standard queue's at-least-once contract makes this non-negotiable.

There is another operational limit that matters during a long interruption: delayed messages can be delayed for at most seven days, retained for at most 30 days, and are deleted when acknowledged. This is work delivery, not a permanent event log. If the business requirement is to replay months of reservation events into multiple independent consumers, a queue cleanup design has already left its natural territory.

Short jobs stay short. Long jobs get chopped into retryable units.

## Build the state transition before the scheduler adapter

The application contract below keeps scheduler behavior outside the domain rule. Cron calls `runExpirySweep`; the function either expires a bounded page directly or publishes identifiers for workers. Both paths reach the same conditional transition. The sample uses interfaces so the database and queue adapters remain replaceable, and it avoids claiming a vendor-specific request body that the application does not need to know.

```ts
interface Reservation {
  id: string;
  holdExpiresAt: Date;
  status: "active" | "expired";
}

interface ReservationStore {
  findDue(now: Date, limit: number): Promise<Reservation[]>;
  expireIfDue(id: string, now: Date): Promise<"expired" | "already-final">;
}

interface WorkQueue {
  publish(message: { reservationId: string }): Promise<void>;
}

type SweepMode = "inline" | "queued";

export async function runExpirySweep(
  store: ReservationStore,
  queue: WorkQueue,
  now: Date,
  mode: SweepMode,
): Promise<{ found: number; completed: number; enqueued: number }> {
  const due = await store.findDue(now, 500);
  let completed = 0;
  let enqueued = 0;

  for (const reservation of due) {
    if (mode === "queued") {
      await queue.publish({ reservationId: reservation.id });
      enqueued += 1;
      continue;
    }

    const result = await store.expireIfDue(reservation.id, now);
    if (result === "expired") completed += 1;
  }

  return { found: due.length, completed, enqueued };
}

export async function consumeExpiryWork(
  store: ReservationStore,
  message: { reservationId: string },
  now: Date,
): Promise<void> {
  await store.expireIfDue(message.reservationId, now);
}
```

The concrete SQL behind `expireIfDue` should combine the ID, active status, and elapsed hold deadline in one conditional update. That is the important operation. If the same message arrives twice, the second call returns `already-final` and acknowledges cleanly; it does not reverse or repeat the business transition. The fixed page size of 500 is an example application choice, not a capacity claim. Measure it against the internal run budget, then tune it.

Infrai is one reasonable combined option because its plain REST contract can stay fixed while the vendor behind the capability changes, and the same key covers cron and queue access. The TypeScript below makes a real scheduling call without guessing any write fields: it lists configured cron jobs through the verified route. Pass the API origin through deployment configuration so an unlinked comparison does not pin a vendor URL.

```ts
const apiOrigin = process.env.INFRAI_API_ORIGIN;
const apiKey = process.env.INFRAI_API_KEY;

if (!apiOrigin || !apiKey) {
  throw new Error("Set INFRAI_API_ORIGIN and INFRAI_API_KEY");
}

const sleep = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function listCronJobs(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${apiOrigin}/v1/cron/list`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const waitMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await sleep(waitMs);
      continue;
    }

    if (!response.ok) {
      const reason = await response.text();
      throw new Error(`Cron list failed (${response.status}): ${reason}`);
    }

    return response.json();
  }

  throw new Error("Cron list exhausted its retry budget");
}

console.log(await listCronJobs());
```

The production adapter should use the discovered `POST /v1/cron/create` contract and add an idempotency key for that write. The same retry and error rules apply. Those details belong in the adapter, so they don't leak into the reservation rule above.

## Follow the handoff points in order

Stick with a plain cron sweep when the cleanup is predictably short, the age-window query is indexed, and rerunning that bounded window is the recovery plan you actually want. Adding workers in that case creates a queue to monitor, duplicate delivery to handle, and another place for stale work to wait. Complexity has carrying cost. For a solo founder, it should buy a specific recovery property.

Choose the cron-triggered queue when work must be chunked, individual items need retries, or processing safely needs limited concurrency. The catch is that the push target must be public HTTPS, standard consumers must be idempotent, and this queue is not suitable for Kafka-style replay or multiple consumer groups. It also has no native topic fan-out, debounce, or throttle; separate queues and application logic would be required.

Temporal or Airflow is the better category when reservation cleanup is really a workflow with a DAG, joins, and coordinated steps. Kafka is the better category when retained replay and multiple consumer groups are requirements. AWS SQS deserves evaluation when its visibility-timeout model matches the queue recovery behavior your team already operates. For in-process Node.js queue workers, BullMQ is another alternative to evaluate; Celery fills a comparable worker role in Python systems. None of those options is an automatic upgrade. They solve broader coordination or event-delivery problems, and that breadth is overhead when the job is one indexed expiry transition.

The decision rule I would put in the runbook is concrete: begin with an age-based cron sweep; move the per-record transition behind idempotent queue consumers before the worst-case run threatens the time budget or before whole-batch retries become operationally expensive. If you need joins or replay, skip that incremental step and choose the category built for it.

## References

- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html
- https://www.rfc-editor.org/rfc/rfc2104
- https://airflow.apache.org/docs/
- https://docs.temporal.io/
- https://kafka.apache.org/documentation/
- https://docs.bullmq.io/
- https://docs.celeryq.dev/

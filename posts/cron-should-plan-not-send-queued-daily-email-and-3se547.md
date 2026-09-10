# Cron Should Plan, Not Send: Queued Daily Email and Idempotent Retries

Short answer: a daily scheduled email handler should enqueue one durable job per shipment update and return; workers should send the emails with stable idempotency keys, bounded retries, and a dead-letter path.

That boundary turns a large, ambiguous cron run into small recovery units. If subscriber 4,812 is rate-limited, the system retries subscriber 4,812. It doesn't restart every delivery that came before it. The scheduler records what should happen today; workers carry out each side effect and leave evidence behind.

The operating rule is crisp: **the cron run is successful when today's work is durably accepted, not when every email has been sent**. Delivery success is a separate metric.

## Why move the send out of the cron handler?

Picture the inline version first. At 09:00, cron loads all subscribers for a shipment, renders an update, calls the mail service, and repeats until the list ends. This looks direct. It also puts recipient lookup, rendering, network calls, rate limiting, and completion tracking inside one execution. A failure late in the loop leaves an awkward question: which earlier calls took effect?

Now change just the boundary. At 09:00, cron identifies the shipment update and enqueues delivery jobs. Workers consume those jobs at a controlled rate. Each job carries a stable identity, and each completed send produces a durable delivery record. The diagram in words is: **clock -> planner -> queue -> workers -> mail provider**, with the delivery store beside the workers.

Small units matter.

| Concern | Send inside cron | Enqueue, then send in workers |
| --- | --- | --- |
| Retry unit | Entire scheduled run | One subscriber delivery |
| Rate limit | Stalls the scheduling loop | Delays affected work |
| Completion signal | Handler exit can hide partial delivery | Planner and delivery have separate states |
| Recovery evidence | Loop position and logs | Durable key, attempt state, and provider receipt |

Suppose a shipment update has 10,000 subscribers. The inline handler successfully asks the provider to send 7,203 messages, loses its connection while processing the next subscriber, and never reaches the rest. Retrying the whole handler risks revisiting 7,203 completed requests; abandoning it skips the remaining subscribers. An enqueue-first design narrows the uncertainty to one job. Completed jobs stay complete, waiting jobs remain visible as backlog, and the uncertain job can be checked using its stable identity before another external side effect is attempted. This isn't a promise of magical exactly-once delivery. It is a way to make duplicate execution safe and recovery observable.

The distinction also fixes misleading monitoring. A green cron status means planning succeeded. It does not mean the mailing finished. Track at least the number of jobs planned, accepted, processing, retried, delivered, and moved aside for investigation. Then alert on age as well as count: a flat queue of recent work may be healthy, while one old job can reveal a subscriber-specific poison case.

## How should a daily scheduled email enqueue jobs for idempotent retries?

Use a deterministic key based on the business event, not on an individual attempt. For this example, `shipmentId + updateVersion + subscriberId` identifies the intended delivery. A retry keeps that key. Tomorrow's genuinely new update gets a different version and therefore a different key.

The planner should also have its own stable run identity. A repeated cron invocation for the same shipment update can safely propose the same jobs if the enqueue ledger enforces uniqueness. That covers a common recovery case: the planner loses its response after persisting some work and is invoked again. Don't create random delivery IDs inside the retry loop; random IDs disguise duplicates as new work.

Here is a compact TypeScript shape. The interfaces are intentionally generic. Queue and email products differ, but the ownership of each state transition should not.

```ts
type DeliveryJob = {
  shipmentId: string;
  updateVersion: string;
  subscriberId: string;
  email: string;
};

type DeliveryState = "sending" | "sent" | "retryable";

interface DeliveryStore {
  begin(key: string): Promise<{ state: DeliveryState }>;
  markSent(key: string, providerMessageId: string): Promise<void>;
  markRetryable(key: string, retryAt: Date): Promise<void>;
}

interface MailGateway {
  send(input: {
    to: string;
    shipmentId: string;
    idempotencyKey: string;
  }): Promise<{ providerMessageId: string }>;
}

const deliveryKey = (job: DeliveryJob): string =>
  `${job.shipmentId}:${job.updateVersion}:${job.subscriberId}`;

async function deliver(
  job: DeliveryJob,
  store: DeliveryStore,
  mail: MailGateway,
): Promise<void> {
  const key = deliveryKey(job);
  const claim = await store.begin(key);

  if (claim.state === "sent") return;

  try {
    const result = await mail.send({
      to: job.email,
      shipmentId: job.shipmentId,
      idempotencyKey: key,
    });
    await store.markSent(key, result.providerMessageId);
  } catch (error) {
    const retryAt = new Date(Date.now() + retryDelayMs(error));
    await store.markRetryable(key, retryAt);
    throw error;
  }
}

function retryDelayMs(error: unknown): number {
  // Parse a valid Retry-After value when the gateway exposes one.
  // Otherwise, return a bounded backoff delay with jitter.
  return 30_000 + Math.floor(Math.random() * 5_000);
}
```

The storage operation behind `begin` needs an atomic uniqueness rule on the delivery key. The provider call should receive that same key when its API supports idempotency. These two controls cover different gaps: the store stops known completed work, while provider-side idempotency helps when a request may have succeeded but its response was lost.

There is still an unavoidable edge if the provider offers no idempotency mechanism: a process can stop after the provider accepts mail but before `markSent` commits. I'm not sure any generic wrapper can remove that ambiguity without a provider receipt lookup or an equivalent reconciliation API. Be honest about it. Keep the state as uncertain, reconcile before resending when possible, and measure these cases separately from ordinary failures.

Uncertainty belongs in state — not in an operator's memory.

## What should happen after a rate limit or failed attempt?

HTTP `429 Too Many Requests` means the client has sent too many requests in a given period. A response may include `Retry-After` to indicate how long to wait before another request. A worker should honor that signal when present, reduce pressure, and reschedule the individual delivery rather than hold a thread open or fail the whole shipment update.

Retries need a ceiling. Use exponential backoff with jitter for transient failures, cap the delay, and cap the attempt count or elapsed retry window. The exact values depend on the mail provider and the usefulness window of the shipment update; your mileage may vary. A delayed update can become less helpful than no update, so encode an expiry policy instead of retrying forever.

After the retry policy is exhausted, move the job to a dead-letter queue or equivalent quarantine. AWS documents dead-letter queues as targets for messages that could not be processed successfully, along with a redrive policy that controls when source-queue messages move there. Treat that destination as an operational inbox, not a trash can. Preserve the delivery key, attempt count, last classified error, and correlation identifier. Alert on arrivals. Provide a reviewed redrive action after the underlying cause has been addressed.

One caveat is important: the AWS documentation warns against using a dead-letter queue when exact message order must be preserved. If shipment notifications have strict sequence semantics, blindly removing one failed update can let a later status overtake it. Partition work by shipment or subscriber, stop that partition on failure, or design the email content so each update stands on its own. Stick with ordered, blocked processing when sequence correctness matters more than throughput.

## Can the queue create duplicate daily report emails?

Yes. Duplicate execution is a normal condition to design for, even if a particular queue offers de-duplication controls. The durable business key is the defense that survives delayed retries, operator redrives, and repeated scheduling. Queue-level de-duplication can help, but it should not be the only guard around an external side effect.

This is also why a job should contain identifiers and immutable version references rather than a pre-rendered assumption about current database state. The worker must know which shipment update it was asked to send. If it silently reads “latest,” a retry tomorrow could send different content under yesterday's delivery key. Either store the immutable content snapshot or bind the job to a version that can be rendered deterministically.

Don't acknowledge early. A worker acknowledges only after the durable record captures the successful provider result. If the job expires, is suppressed because the subscriber opted out, or is otherwise intentionally not sent, record that terminal decision explicitly. “Missing from the queue” is not evidence of delivery.

## The operational checklist before deployment

Test recovery, not just the happy path. Invoke the planner twice for the same update and verify that it produces one logical job per subscriber. Deliver the same job twice and verify that the second execution sees the completed key. Simulate a 429 response and confirm that only the affected job is delayed. Exhaust the retry budget, inspect the quarantined payload, then redrive it through the same idempotent consumer.

A useful drill starts with a deliberately mixed batch: one ordinary delivery, one duplicate of that delivery, one response carrying HTTP 429 and `Retry-After`, and one error classified as non-retryable by the gateway adapter. Run the batch, restart a worker between the provider call and its local completion step, and then inspect state without reading application logs first. The ordinary job should be terminal, the duplicate should resolve against the same delivery key, the rate-limited job should have a future attempt time, and the non-retryable job should be available for review. The interrupted job is the hard one; its next action must follow the provider's idempotency or reconciliation contract. This drill tests the recovery model itself — logs merely explain what the durable state already says.

Deploy with separate dashboards for planning and delivery. The planner needs run duration, recipient count, enqueue failures, and a run identifier. Workers need queue age, attempt counts, throughput, delivery outcomes, and dead-letter arrivals. Correlation should connect a shipment update to a subscriber delivery without putting unnecessary personal data into logs.

The trade-off is real. A queue adds infrastructure, asynchronous debugging, backlog capacity planning, and an operator-owned recovery path. It is not suitable when a cron task has one quick, local, reversible action and retrying the entire task is already safe. Keep that simple task inline. For a fan-out shipment update, however, the queue earns its place because the recovery unit matches the side effect: one intended email.

Choose the boundary before choosing a queue product. The durable design is the same: schedule intent once, execute bounded jobs, make retries idempotent, and expose enough state for an operator to recover without guessing.

## References

- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429

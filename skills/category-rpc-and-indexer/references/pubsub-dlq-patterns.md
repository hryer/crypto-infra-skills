# GCP Pub/Sub + DLQ Patterns for Blockchain Indexers

Concrete patterns for using GCP Pub/Sub as the durable buffer in a blockchain indexer, with DLQ.

## Topology

```
Webhook handler
      │
      ▼
[blockchain-events]  (main topic)
      │
      ▼
[blockchain-events-sub]  (subscription with DLQ policy)
      │
      ├──> Worker (consumer)  ──> processes successfully
      │
      └──> After N failures
            │
            ▼
      [blockchain-events-dlq]  (dead letter topic)
            │
            ▼
      [blockchain-events-dlq-sub]  (manual operator review)
```

## Topic + subscription setup (Terraform)

```hcl
resource "google_pubsub_topic" "events" {
  name = "blockchain-events"
}

resource "google_pubsub_topic" "events_dlq" {
  name = "blockchain-events-dlq"
}

resource "google_pubsub_subscription" "events_sub" {
  name  = "blockchain-events-sub"
  topic = google_pubsub_topic.events.id

  ack_deadline_seconds = 60

  retry_policy {
    minimum_backoff = "1s"
    maximum_backoff = "600s"  # 10 min cap
  }

  dead_letter_policy {
    dead_letter_topic     = google_pubsub_topic.events_dlq.id
    max_delivery_attempts = 5
  }

  expiration_policy {
    ttl = ""  # never expire
  }
}

resource "google_pubsub_subscription" "events_dlq_sub" {
  name  = "blockchain-events-dlq-sub"
  topic = google_pubsub_topic.events_dlq.id

  ack_deadline_seconds = 600  # 10 min for human review
}
```

## Webhook handler (NestJS example)

```typescript
import { Controller, Post, Headers, Body, HttpCode } from '@nestjs/common';
import { PubSub } from '@google-cloud/pubsub';

@Controller('webhooks/alchemy')
export class AlchemyWebhookController {
  private pubsub = new PubSub();
  private topic = this.pubsub.topic('blockchain-events');

  @Post()
  @HttpCode(200)  // ack fast
  async handle(
    @Headers('x-alchemy-signature') sig: string,
    @Body() body: any,
  ) {
    // 1. Verify signature (fast, in-memory)
    if (!this.verifySignature(body, sig)) {
      throw new Error('invalid signature');
    }

    // 2. Push to Pub/Sub. Do NOT process inline.
    const messageId = await this.topic.publishMessage({
      json: body,
      attributes: {
        chain: body.event.network,
        receivedAt: new Date().toISOString(),
        webhookId: body.webhookId,
      },
    });

    // 3. Return 200 immediately. Total time: <100ms.
    return { messageId };
  }

  private verifySignature(body: any, sig: string): boolean {
    // ... HMAC verification
  }
}
```

## Worker (consumer)

```typescript
import { PubSub, Message } from '@google-cloud/pubsub';

const subscription = new PubSub().subscription('blockchain-events-sub');

subscription.on('message', async (message: Message) => {
  const event = JSON.parse(message.data.toString());
  const idempotencyKey = `${event.chainId}:${event.txHash}:${event.logIndex}`;

  try {
    // 1. Hot dedup via Redis
    const isNew = await redis.set(
      `processed:${idempotencyKey}`,
      '1',
      'NX',
      'EX',
      86400,  // 24h TTL
    );
    if (!isNew) {
      // Already processed. Ack and move on.
      message.ack();
      return;
    }

    // 2. Process the event
    await processor.handle(event);

    // 3. Cold dedup via DB (UNIQUE constraint)
    await db.insert('processed_events', {
      idempotency_key: idempotencyKey,
      processed_at: new Date(),
    });

    message.ack();
  } catch (err) {
    if (err.code === '23505') {
      // UNIQUE violation = race condition, already processed by another worker
      message.ack();
      return;
    }

    // Real error. nack() triggers retry per the subscription's retry policy.
    // After max_delivery_attempts (5), Pub/Sub moves the message to the DLQ.
    logger.error({ err, event }, 'processing failed');
    message.nack();
  }
});
```

## DLQ consumer (operator review)

```typescript
const dlqSub = new PubSub().subscription('blockchain-events-dlq-sub');

dlqSub.on('message', async (message: Message) => {
  const event = JSON.parse(message.data.toString());
  const deliveryAttempt = message.deliveryAttempt;

  // Don't auto-retry. Log + alert.
  await alerting.send({
    severity: 'high',
    title: `Indexer DLQ: ${event.chainId} ${event.txHash}`,
    payload: event,
    attempts: deliveryAttempt,
  });

  // Persist to a dlq_events table for human review
  await db.insert('dlq_events', {
    event_data: event,
    received_at: new Date(),
    attempts: deliveryAttempt,
  });

  // Ack so it doesn't keep redelivering
  message.ack();
});
```

## Operator runbook

When a DLQ alert fires:

1. **Look at the persisted error.** What failed? Bug in code? Bad data? Schema change?
2. **Categorize:**
   - **Code bug** — fix code, then replay all DLQ entries that hit the same bug
   - **Bad data** — discard (or escalate to vendor if the data was malformed at source)
   - **Transient infra** — should have been handled by retries; investigate why it wasn't
3. **Replay procedure:**
   - Pull DLQ entries from the `dlq_events` table
   - Re-publish them to the main topic with a `replayed=true` attribute
   - Confirm they process successfully
   - Mark the dlq_events row as resolved

## Key configurations

| Setting | Recommendation | Reason |
|---|---|---|
| `ack_deadline_seconds` | 60 | Long enough for real processing, short enough to recover from worker crashes |
| `max_delivery_attempts` | 5 | More than 5 retries on a poison message is wasteful |
| `minimum_backoff` | 1s | Fast first retry handles transient blips |
| `maximum_backoff` | 600s | Bounded so retries don't drag forever |
| Worker concurrency | 10-50 per pod | Tune based on processor throughput |
| Subscription type | Pull (push only for very-low-volume scenarios) | Pull gives you flow control |

## Anti-patterns

- **Don't process inside the webhook handler.** Webhook = pub. Worker = sub.
- **Don't have one giant subscription for all chains.** One sub per chain lets you scale workers independently and isolate failures.
- **Don't ack before processing succeeds.** Ack means "delete this message forever." Only ack after the DB write commits.
- **Don't nack messages indefinitely.** That's what the DLQ is for. After 5 attempts, let it go.
- **Don't ignore the deliveryAttempt counter.** A message at attempt 4 deserves more scrutiny than attempt 1.

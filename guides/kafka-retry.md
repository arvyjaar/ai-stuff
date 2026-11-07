# Implementing retries with Kafka

## Sources

- [Uber: Building Reliable Reprocessing and Dead Letter Queues with Kafka](https://www.uber.com/us/en/blog/reliable-reprocessing/)
- [Spring Kafka: non-blocking retries (`@RetryableTopic`)](https://docs.spring.io/spring-kafka/reference/retrytopic/retry-config.html)

## The three layers

Most teams combine three layers:

```
main topic ──fail (temporary)──► retry ──fail again──► retry ... ──retries used up──► DLQ
     │                              │                                                    ▲
     └──fail (permanent)────────────┼────────────────────────────────────────────────────┘
                                    └──success──► done
```

### 1. Short retry in place (blocking)

Retry inside the consumer a few times with a short backoff, for example 3 tries over about 1 second. This handles brief network problems without any extra topics.

Keep it short: the partition is blocked while the consumer retries.

### 2. Retry topics with fixed delays (non-blocking)

This is the pattern Uber describes, and it's what Spring Kafka's `@RetryableTopic` does by default. There is one topic **per delay level**:

```
orders → orders-retry-10s → orders-retry-1m → orders-retry-10m → orders-dlq
```

- The original message is committed only **after** the copy has been successfully written to the next topic.
- Each retry topic's consumer waits until a message is due before processing it.

**The fixed delay per topic is the important part.** Every message in `orders-retry-1m` is due one minute after it was written, so the messages are already in due order. Waiting for the first one never holds up a message that's due sooner.

> **Anti-pattern:** a single retry topic with growing delays (e.g. `factor * retryCount`) and a `time.Sleep` per message. A message due in 50 seconds then holds up one due in 5 seconds. The ovoko `libraries/go/kafka` v1.3.0 `RetryConsumer` works this way.

### 3. Dead-letter queue (DLQ) with alerting and replay

| Part | What it means |
|---|---|
| Storage | A Kafka topic, or a database table if you want to query and edit messages |
| Context | Original topic, partition, offset, error, attempt count and timestamp, usually as headers |
| Alerting | Alert on any DLQ message, or on a growing count |
| Replay tool | Sends selected messages back to the main topic after the fix. Without one, a DLQ just collects messages nobody reprocesses |

Other systems have the same idea built in:

- **Kafka Connect:** `errors.tolerance=all` + `errors.deadletterqueue.topic.name`
- **AWS SQS:** redrive policy with `maxReceiveCount`

## Rules almost everyone follows

1. **Never commit until the message is processed or safely moved** to a retry topic or the DLQ. If that move fails, don't commit: stop or retry.
2. **Handlers must be safe to run twice.** Retries and redelivery mean duplicates. Use idempotency keys, upserts, or deterministic workflow IDs that reject duplicates.
3. **Non-blocking retry breaks per-key ordering.** While message #2 for payout X waits in retry, message #3 for payout X gets processed. Common fixes:
   - Make the handler reject outdated updates, for example a status transition that moves backwards. This is the most common fix, and usually the best one.
   - Use blocking retry for topics where order matters.
   - Track keys that have a message in retry, and route later messages for those keys to retry too, so they stay in order.
4. **Monitor** consumer lag, retry topic sizes and DLQ count.

## Simpler alternative: DLQ + manual replay

This setup has no retry topics. Failed messages go to a DLQ; you fix the code or data, then run a command that publishes them back to the main topic.

### Sort errors by cause

| Failure | Example | Action |
|---|---|---|
| **The message is the problem** (permanent) | Bad JSON, failed validation, missing data | Send to DLQ immediately, then commit |
| **A dependency is the problem** (temporary) | DB down, Temporal unavailable, timeout | Retry in place with backoff (blocks). After a cap (e.g. 5–10 min), send to DLQ |

Blocking during an outage costs nothing, because the next message would fail as well. Once the dependency is back, the consumer continues by itself and there's nothing to replay. Without this, an outage would push every message into the DLQ.

```
fetch → process
          ├─ ok ──────────────────────────────► commit
          ├─ temporary ─► retry in place (blocks)
          │                 ├─ recovers ──────► commit
          │                 └─ cap reached ───► DLQ ─► commit
          └─ permanent ───────────────────────► DLQ ─► commit
                         if the DLQ write fails ─► don't commit, stop
```

### Replay rules

> [!IMPORTANT]
> **Replay with the same message key.** Kafka chooses the partition from the key. If the key is different or missing, the message can land on another partition and be processed out of order with, or in parallel with, other messages for the same entity (payout, transfer, order). Nothing errors when this happens, so it's easy to miss.

- **Expect the message to arrive after newer ones.** A replayed message goes to the end of the topic. Handlers must be safe to run twice and must reject outdated updates; for example, an old "created" must not overwrite "succeeded".
- **Keep the original value and headers** (including trace IDs). Change only the DLQ metadata below.
- **Filter and dry-run:** replay only what was fixed, e.g. `replay --topic payouts --key X --since ... --dry-run`.
- **Commit the DLQ position only after the publish succeeds.**
- **Alert** when the DLQ is not empty, and on consumer lag.

### DLQ metadata (headers)

Yes, add metadata, but keep it small. Overwrite these headers instead of appending, so repeated failures don't stack duplicates.

| Header | Set by | Why |
|---|---|---|
| `dlq-original-topic`, `dlq-original-partition`, `dlq-original-offset` | Consumer, when writing to DLQ | Find the original message; tells the replay command where to publish |
| `dlq-error`, `dlq-failed-at` | Consumer, when writing to DLQ | Lets you group and filter failures without reading logs |
| `dlq-replay-count` | Replay command (+1) | If a replayed message fails again, it returns to the DLQ with count ≥ 1. That means the fix didn't work, so stop replaying it |

Handlers can log `dlq-replay-count`, but must **not** change business logic based on it. A replayed message is processed exactly like a new one.

A DLQ **database table** is often easier than a topic for this manual workflow. You can query it with SQL and track each message's status (`new` / `replayed` / `ignored`).

## RabbitMQ vs Kafka

|  | RabbitMQ | Kafka |
|---|---|---|
| **Model** | Queue: messages removed after ack | Log: messages kept, each group tracks its position |
| **Per-message retry and DLQ** | Built in | You build it (retry topics + DLQ topic) |
| **Delayed retry** | Workaround (TTL + DLX, or plugin) | Workaround (fixed-delay retry topics) |
| **Ordering** | Weak once you retry or scale out | Strong per partition |
| **Replay / reprocessing** | No | Yes |
| **Many independent consumers of one stream** | Copy into each queue | Natural (consumer groups) |
| **Throughput** | High | Very high |
| **Best fit** | Task queues, commands, jobs | Event streams, event sourcing, many subscribers |

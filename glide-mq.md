# Learning Guide: glide-mq

**Generated**: 2026-03-19
**Sources**: 12 resources analyzed
**Depth**: medium

## Prerequisites

What you should know before diving in:
- Node.js async/await and EventEmitter patterns
- Basic understanding of message queues (producer/consumer, at-least-once delivery)
- Valkey or Redis fundamentals (keys, streams, consumer groups)
- If migrating: familiarity with BullMQ API

## TL;DR

- glide-mq is a high-performance Node.js message queue built on Valkey/Redis Streams with a Rust NAPI core (via valkey-glide)
- All queue operations execute as a single Valkey Server Function (FCALL) - no Lua EVAL overhead, 1 round-trip per job
- Cluster-native with hash-tagged keys (`glide:{queueName}:*`), cloud-ready with AZ-affinity and IAM auth
- 35-38% faster than BullMQ at concurrency >= 10 (benchmarked on ElastiCache r7g.large)
- API mirrors BullMQ closely - Queue, Worker, FlowProducer, QueueEvents - making migration straightforward

## Core Concepts

### Architecture

glide-mq sits on three layers:

1. **Valkey Server Functions** - A single persistent function library loaded once via `FUNCTION LOAD`. All queue operations (enqueue, dequeue, acknowledge, delay, priority sort) happen server-side in one `FCALL` round-trip. No per-call Lua recompilation.

2. **Rust NAPI Client** - Built on `@valkey/valkey-glide`, the official Valkey client. Protocol parsing and connection management happen in Rust, skipping JavaScript overhead. Less GC pressure than ioredis-based alternatives.

3. **Node.js API** - TypeScript classes (Queue, Worker, Producer, FlowProducer, Broadcast) with EventEmitter lifecycle hooks. API design closely mirrors BullMQ for easy migration.

**Key insight**: The combination of server-side functions + native client means glide-mq eliminates both the multi-roundtrip overhead of Lua scripts AND the JavaScript protocol parsing overhead. This is why the performance gap widens at higher concurrency.

### Job Lifecycle

Jobs flow through states: `waiting` -> `active` -> `completed` | `failed` -> (retry) -> `waiting` | `dead-letter`.

The `completeAndFetchNext` pattern finishes the current job and claims the next one in a single FCALL, halving the round-trips per job compared to separate acknowledge + fetch.

### Cluster-Native Design

All keys for a queue use the hash tag pattern `glide:{queueName}:*`. This forces Valkey Cluster to co-locate all queue data on the same shard - no cross-slot errors, no manual slot management. Works identically on standalone and cluster deployments.

## Code Examples

### Basic Producer/Consumer

```typescript
import { Queue, Worker } from 'glide-mq';

const connection = { addresses: [{ host: 'localhost', port: 6379 }] };

// Producer
const queue = new Queue('emails', { connection });
await queue.add('welcome', { to: 'user@example.com', template: 'onboarding' });

// Consumer
const worker = new Worker('emails', async (job) => {
  await sendEmail(job.data.to, job.data.template);
  return { sent: true };
}, { connection, concurrency: 10 });

worker.on('completed', (job) => console.log(`Sent: ${job.id}`));
worker.on('failed', (job, err) => console.error(`Failed: ${job.id}`, err.message));
```

### Delayed & Priority Jobs

```typescript
// Process in 5 minutes
await queue.add('reminder', data, { delay: 300_000 });

// High priority (higher number = earlier execution)
await queue.add('urgent-alert', data, { priority: 10 });

// Retries with exponential backoff
await queue.add('webhook', data, {
  attempts: 5,
  backoff: { type: 'exponential', delay: 1000 }
});
```

### Bulk Ingestion

```typescript
// 10,000 jobs in ~350ms
const jobs = users.map(u => ({
  name: 'sync-profile',
  data: { userId: u.id },
  opts: { jobId: `sync-${u.id}` } // Idempotent
}));
await queue.addBulk(jobs);
```

### Batch Processing

```typescript
const worker = new Worker('analytics', async (jobs) => {
  // jobs is Job[] - process in bulk
  const rows = jobs.map(j => j.data);
  await db.insertMany('events', rows);
}, {
  connection,
  batch: { size: 50, timeout: 5000 } // Collect 50 jobs or wait 5s
});
```

### Workflows (Parent-Child DAGs)

```typescript
import { FlowProducer } from 'glide-mq';

const flow = new FlowProducer({ connection });
await flow.add({
  name: 'process-order',
  queueName: 'orders',
  data: { orderId: 123 },
  children: [
    { name: 'charge-payment', queueName: 'payments', data: { amount: 99 } },
    { name: 'reserve-inventory', queueName: 'inventory', data: { sku: 'X' } }
  ]
});
// Parent waits for all children to complete
```

### Broadcast (Fan-Out Pub/Sub)

```typescript
import { Broadcast, BroadcastWorker } from 'glide-mq';

const broadcast = new Broadcast('notifications', { connection });
await broadcast.add('order.completed', { orderId: 123 });

// Each subscriber group gets independent delivery + retries
const emailWorker = new BroadcastWorker('notifications', 'email-svc',
  async (msg) => await sendNotificationEmail(msg.data),
  { connection, concurrency: 5 }
);

const slackWorker = new BroadcastWorker('notifications', 'slack-svc',
  async (msg) => await postToSlack(msg.data),
  { connection }
);
```

### Schedulers (Cron Jobs)

```typescript
import { Scheduler } from 'glide-mq';

const scheduler = new Scheduler(queue);
await scheduler.addCron('daily-report', { type: 'summary' }, {
  pattern: '0 9 * * *',
  tz: 'America/New_York'
});

await scheduler.addRepeatableJob('health-check', {}, {
  interval: 60_000,
  limit: 1440 // Max 1 day of runs
});
```

### Serverless (Lambda/Vercel Edge)

```typescript
import { ServerlessPool } from 'glide-mq';

const pool = new ServerlessPool(connection);

export async function handler(event) {
  const queue = pool.getQueue('tasks');
  await queue.add('process', event.body);
  return { statusCode: 202 };
}
// Connection reused across warm invocations
```

### Lightweight Producer (No EventEmitter)

```typescript
import { Producer } from 'glide-mq';

const producer = new Producer(connection);
const jobId = await producer.add('queue-name', 'job-name', data);
await producer.close();
// Returns plain string ID, no Job object instantiated
```

### Testing Without Valkey

```typescript
import { TestQueue, TestWorker } from 'glide-mq/testing';

const queue = new TestQueue('tasks');
await queue.add('test-job', { key: 'value' });

const worker = new TestWorker(queue, async (job) => {
  expect(job.data.key).toBe('value');
  return { ok: true };
});
await worker.run();
```

## Connection & Configuration

### Connection Options

```typescript
const connection = {
  addresses: [{ host: 'valkey.example.com', port: 6379 }],
  clientName: 'my-service',
  tlsMode: 'SecureConnection',       // TLS
  credentials: { username: 'default', password: 'secret' },
  readFrom: 'AZAffinity',            // Route reads to same-AZ replica
  protocol: 7                         // Valkey 7+
};
```

### Default Job Options

```typescript
const queue = new Queue('tasks', {
  connection,
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 1000 },
    compress: true,   // Gzip payloads >1KB (98% reduction on 15KB)
    ttl: 86_400_000   // 24h expiration
  },
  rateLimit: { maxCount: 1000, interval: 1000 } // 1000 jobs/sec
});
```

### Custom Serializers

```typescript
import { msgpackSerialize, msgpackDeserialize } from 'msgpackr';

const queue = new Queue('tasks', {
  connection,
  serializer: {
    serialize: msgpackSerialize,
    deserialize: msgpackDeserialize
  }
});
```

## Observability

### OpenTelemetry

```typescript
import * as api from '@opentelemetry/api';

const queue = new Queue('tasks', {
  connection,
  instrumentationConfig: { tracer: api.trace.getTracer('app') }
});
// Automatic span emission for all queue operations
```

### Time-Series Metrics & Job Logs

```typescript
const metrics = await queue.getMetrics(60); // Last 60 minutes

// Per-job structured logs
await job.log('Step 1', { duration: 523 });
const logs = await job.getLogs(0, 10);
```

### QueueEvents (Real-Time Stream)

```typescript
const events = new QueueEvents('tasks', { connection });
events.on('completed', ({ jobId }) => console.log(`Done: ${jobId}`));
events.on('failed', ({ jobId, failedReason }) => console.error(failedReason));
events.on('stalled', ({ jobId }) => console.warn(`Stalled: ${jobId}`));
```

## Framework Integrations

| Framework | Package | Pattern |
|-----------|---------|---------|
| Hono | `@glidemq/hono` | Middleware |
| Fastify | `@glidemq/fastify` | Plugin |
| NestJS | `@glidemq/nestjs` | Module |
| Hapi | `@glidemq/hapi` | Plugin |
| Dashboard | `@glidemq/dashboard` | Express middleware |

All integrations provide REST endpoints, SSE streams, and serverless Producer support.

## Common Pitfalls

| Pitfall | Why It Happens | How to Avoid |
|---------|---------------|--------------|
| Jobs processed twice after crash | At-least-once semantics - unacked jobs re-deliver | Make processors idempotent; use `jobId` for dedup |
| Stalled jobs | Worker dies without releasing lock | Set `maxStalledCount` and `lockDuration` appropriately |
| OOM on large payloads | Job data stored in Valkey memory | Enable `compress: true`; keep payloads small, store large data externally |
| Cross-slot errors on cluster | Keys not hash-tagged | Always use glide-mq's built-in key naming (don't manually create keys) |
| Cold start latency in serverless | New connection per invocation | Use `ServerlessPool` to cache connections across warm starts |
| Rate limit not working | Applied per queue instance, not globally | Use server-side `rateLimit` option on Queue constructor |

## Best Practices

Synthesized from multiple sources:

1. **Use `addBulk` for batch ingestion** - 12.7x faster than serial `add()` calls. Always batch when adding multiple jobs.
2. **Set concurrency based on workload** - CPU-bound: match core count. IO-bound: 10-50x core count. The 35% speedup over BullMQ appears at concurrency >= 10.
3. **Enable compression for payloads >1KB** - 98% reduction on 15KB payloads with negligible CPU cost.
4. **Use `Producer` in serverless** - Lighter than `Queue` (no EventEmitter, no worker overhead).
5. **Idempotent processors + custom `jobId`** - At-least-once delivery means your processor MUST handle re-delivery gracefully.
6. **Use `UnrecoverableError` for poison messages** - Skips all retries and fails immediately instead of burning retry budget.
7. **Graceful shutdown** - Always call `gracefulShutdown([...])` to drain active jobs before process exit.
8. **Use `TestQueue`/`TestWorker` in CI** - No Valkey instance needed for unit tests.

## Migration from BullMQ

The API is intentionally similar. Key differences:

| BullMQ | glide-mq | Notes |
|--------|----------|-------|
| `new Queue(name, { connection: { host, port } })` | `new Queue(name, { connection: { addresses: [{ host, port }] } })` | Connection format differs (valkey-glide style) |
| Uses ioredis | Uses @valkey/valkey-glide | Rust NAPI instead of JS client |
| Lua EVAL scripts | Valkey Server Functions (FCALL) | Single persistent function library |
| `QueueScheduler` (removed in v5) | `Scheduler` | Built-in cron and interval support |
| No broadcast | `Broadcast` / `BroadcastWorker` | Fan-out with independent subscriber retries |
| No batch processing | `batch: { size, timeout }` | Native multi-job processing |

## Limitations

- Requires Valkey 7.0+ or Redis 7.0+ (no embedded/in-memory mode for prod)
- Node.js only - Rust NAPI bindings don't work in browsers or Deno
- At-least-once delivery (not exactly-once)
- Not a streaming platform - for event streaming use Kafka/NATS JetStream
- Native addon compilation required (`@glidemq/speedkey`)

## Further Reading

| Resource | Type | Why Recommended |
|----------|------|-----------------|
| [glide-mq GitHub](https://github.com/avifenesh/glide-mq) | Official Repo | Full README, examples, migration guide |
| [Valkey Streams Intro](https://valkey.io/topics/streams-intro/) | Official Docs | Understand the underlying data structure |
| [Valkey Server Functions](https://valkey.io/topics/functions-intro/) | Official Docs | How FCALL works under the hood |
| [Valkey GLIDE Node.js](https://glide.valkey.io/languages/nodejs/) | Official Docs | The Rust NAPI client glide-mq builds on |
| [@valkey/valkey-glide npm](https://www.npmjs.com/package/@valkey/valkey-glide) | npm | Client library dependency |
| [BullMQ Docs](https://docs.bullmq.io/) | Official Docs | Useful for migration comparison |

---

*Generated by /learn from 12 sources.*
*See `resources/glide-mq-sources.json` for full source metadata.*

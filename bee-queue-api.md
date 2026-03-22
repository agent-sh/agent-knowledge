# Bee Queue API Reference (v2.0.0)

Complete API surface documentation for migration mapping to glide-mq.

**Package:** `bee-queue` (npm)
**Repository:** https://github.com/bee-queue/bee-queue
**Latest version:** 2.0.0
**License:** MIT
**Redis requirement:** 2.8+ (3.2+ recommended)
**Node.js:** Callback and Promise dual API throughout

---

## Table of Contents

- [Queue Constructor and Settings](#queue-constructor-and-settings)
- [Queue Properties](#queue-properties)
- [Queue Methods](#queue-methods)
- [Queue Events (Local)](#queue-events-local)
- [Queue Events (PubSub)](#queue-events-pubsub)
- [Job Properties](#job-properties)
- [Job Methods (Chaining API)](#job-methods-chaining-api)
- [Job Events](#job-events)
- [Health Check](#health-check)
- [Backoff Strategies](#backoff-strategies)
- [Custom Backoff Strategies](#custom-backoff-strategies)
- [Redis Connection Options](#redis-connection-options)
- [Concurrency Model](#concurrency-model)
- [Stall Detection Internals](#stall-detection-internals)
- [Redis Key Layout](#redis-key-layout)
- [Delayed Job Activation](#delayed-job-activation)
- [Bulk Operations](#bulk-operations)
- [Graceful Shutdown Pattern](#graceful-shutdown-pattern)
- [TypeScript Types](#typescript-types)

---

## Queue Constructor and Settings

```js
const Queue = require('bee-queue');
const queue = new Queue(name, settings?);
```

### Signature

```ts
class BeeQueue<T = any> extends EventEmitter {
  constructor(name: string, settings?: QueueSettings);
}
```

### Settings (all optional)

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `prefix` | `string` | `'bq'` | Redis key namespace prefix. All keys are `{prefix}:{name}:{key}`. |
| `stallInterval` | `number` (ms) | `5000` | Window in which workers must report they are not stalling. Higher = less Redis overhead, but slower stall detection. Also affects false-positive probability. |
| `nearTermWindow` | `number` (ms) | `1200000` (20 min) | Window during which delayed jobs are scheduled via `setTimeout`. If all delayed jobs are further out, the Queue double-checks after this window elapses. |
| `delayedDebounce` | `number` (ms) | `1000` | Groups delayed jobs within this window to avoid unnecessary churn for jobs in short succession. |
| `redis` | `object \| string \| RedisClient` | `{ host: '127.0.0.1', port: 6379, db: 0, options: {} }` | Redis connection config. Can be a `node_redis` `ClientOpts` object, a Redis URL string, or an existing `RedisClient` instance. |
| `isWorker` | `boolean` | `true` | Set `false` if this queue will only enqueue jobs (not process them). Skips creating the blocking Redis connection for `brpoplpush`. |
| `getEvents` | `boolean` | `true` | Set `false` if this queue does not need to receive job events via PubSub. Skips creating the subscriber Redis connection. |
| `sendEvents` | `boolean` | `true` | Set `false` if this worker does not need to send job events (progress, succeeded, failed, retrying) back to other queues. |
| `storeJobs` | `boolean` | `true` | Set `false` if this worker does not need to associate events with specific `Job` instances. Reduces memory usage since the internal jobs Map won't grow. |
| `ensureScripts` | `boolean` | `true` | Ensure Lua scripts exist in Redis before running any commands. |
| `activateDelayedJobs` | `boolean` | `false` | Activate delayed jobs once they pass their `delayUntil` timestamp. **Must be enabled on at least one Queue instance** for `fixed` and `exponential` backoff retry strategies to work. |
| `removeOnSuccess` | `boolean` | `false` | Automatically remove successfully completed jobs from Redis. |
| `removeOnFailure` | `boolean` | `false` | Automatically remove failed jobs from Redis. Does NOT remove jobs set to retry unless they exhaust all retries. |
| `quitCommandClient` | `boolean` | `true` (auto) | Whether to `QUIT` the Redis command client on `Queue#close`. Defaults to `true` normally, `false` if an existing `RedisClient` was provided via `redis` setting. |
| `redisScanCount` | `number` | `100` | Value for `SSCAN` `COUNT` parameter used in `Queue#getJobs` for `succeeded` and `failed` job types. |
| `autoConnect` | `boolean` | `true` | If `false`, `queue.connect()` must be called manually to establish Redis connections. |

### Settings Example (all defaults shown)

```js
const queue = new Queue('test', {
  prefix: 'bq',
  stallInterval: 5000,
  nearTermWindow: 1200000,
  delayedDebounce: 1000,
  redis: {
    host: '127.0.0.1',
    port: 6379,
    db: 0,
    options: {},
  },
  isWorker: true,
  getEvents: true,
  sendEvents: true,
  storeJobs: true,
  ensureScripts: true,
  activateDelayedJobs: false,
  removeOnSuccess: false,
  removeOnFailure: false,
  redisScanCount: 100,
  autoConnect: true,
});
```

### Internal Defaults (from `lib/defaults.js`)

Additional internal defaults not exposed in the public settings documentation:

| Default | Value | Description |
|---------|-------|-------------|
| `initialRedisFailureRetryDelay` | `1000` (ms) | First retry period after a `brpoplpush` failure; backoff is exponential. |
| `#close.timeout` | `5000` (ms) | Default timeout for `Queue#close()`. |
| `#process.concurrency` | `1` | Default concurrency for `Queue#process()`. |

---

## Queue Properties

| Property | Type | Description |
|----------|------|-------------|
| `name` | `string` | The queue name passed to the constructor. |
| `keyPrefix` | `string` | The full Redis key prefix: `{prefix}:{name}:`. |
| `jobs` | `Map<string, Job>` | Map of currently tracked jobs (populated when `storeJobs` and `getEvents` are enabled). |
| `paused` | `boolean` | Whether the queue instance is paused. Only `true` while the queue is in the process of closing. |
| `settings` | `object` | The resolved settings (merge of provided settings and defaults). |
| `backoffStrategies` | `Map<string, (job: Job) => number>` | Map of backoff strategy name to delay-computing function. Pre-populated with `immediate`, `fixed`, and `exponential`. Can be extended with custom strategies. |

---

## Queue Methods

### `Queue#createJob(data)`

```ts
createJob<U extends T>(data: U): Job<U>;
```

Creates and returns a new `Job` object with the associated user data. The job is NOT saved to Redis until `Job#save()` is called.

```js
const job = queue.createJob({ x: 2, y: 3 });
```

---

### `Queue#process([concurrency], handler)`

```ts
process<U>(handler: (job: Job<T>) => Promise<U>): void;
process<U>(handler: (job: Job<T>, done: DoneCallback<U>) => void): void;
process<U>(concurrency: number, handler: (job: Job<T>) => Promise<U>): void;
process<U>(concurrency: number, handler: (job: Job<T>, done: DoneCallback<U>) => void): void;
```

Begins processing jobs with the provided handler function.

**Constraints:**
- Can only be called **once** per Queue instance.
- Should never be called on a queue where `isWorker` is `false`.
- Automatically calls `checkStalledJobs` once on first invocation.

**Parameters:**
- `concurrency` (optional): Maximum number of simultaneously active jobs for this processor. Default: `1`.
- `handler`: Function that processes each job. Must either:
  - Return a `Promise` that resolves (success) or rejects (failure), OR
  - Call `done(err)` for failure, `done(null, result)` for success.
  - If handler returns a Promise, `done` callback is ignored.

**Handler signature:**
```ts
type DoneCallback<T> = (error: Error | null, result?: T) => void;

// Promise-based
async (job: Job<T>) => result;

// Callback-based
(job: Job<T>, done: DoneCallback<U>) => void;
```

**Result must be JSON-serializable** (passed through `JSON.stringify`).

```js
// Promise-based (preferred)
queue.process(async (job) => {
  return job.data.x + job.data.y;
});

// Callback-based
queue.process(function (job, done) {
  return done(null, job.data.x + job.data.y);
});

// With concurrency
queue.process(10, async (job) => {
  return await fetchSomething(job.data);
});
```

---

### `Queue#checkStalledJobs([interval], [cb])`

```ts
checkStalledJobs(interval?: number): Promise<number>;
checkStalledJobs(interval: number, cb: (err: Error, numStalled: number) => void): void;
checkStalledJobs(cb: (err: Error, numStalled: number) => void): void;
```

Checks for stalled jobs and re-enqueues them. Returns the number of stalled jobs detected.

**Behavior based on parameters:**
- `cb` only: `cb` is called with the result.
- `interval` only: Sets a timeout to repeat the check every `interval` ms. Returns Promise for first check only.
- `cb` and `interval`: Sets a timeout and calls `cb` each time.

**Best practice:** Set `interval` to be the same or slightly shorter than `stallInterval`. A good system-wide frequency is every 0.5-10 seconds.

**Maximum stall-to-retry delay:** approximately `stallInterval + interval`.

```js
queue.checkStalledJobs(5000, (err, numStalled) => {
  console.log('Stalled jobs detected:', numStalled);
});
```

---

### `Queue#checkHealth([cb])`

```ts
checkHealth(): Promise<HealthCheckResult>;
checkHealth(cb: (counts: HealthCheckResult) => void): void;
```

Returns job counts by state. See [Health Check](#health-check) for return shape.

```js
const counts = await queue.checkHealth();
console.log(counts);
// { waiting: 5, active: 2, succeeded: 100, failed: 3, delayed: 1, newestJob: '108' }
```

---

### `Queue#close([timeout], [cb])`

```ts
close(cb: () => void): void;
close(timeout?: number | null): Promise<void>;
close(timeout: number | undefined | null, cb: () => void): void;
```

Closes the queue's Redis connections. **Idempotent.**

- `timeout` (optional): Milliseconds to wait for active jobs to finish before force-closing. Default: `5000`.
- Active jobs that don't finish within `timeout` will stall and be retried by another worker.

```js
await queue.close(30000);
```

---

### `Queue#connect()`

```ts
connect(): Promise<boolean>;
```

Establishes Redis connections. Only works if `settings.autoConnect` was set to `false`.

```js
const queue = new Queue('example', { autoConnect: false });
await queue.connect();
```

---

### `Queue#ready([cb])`

```ts
ready(): Promise<this>;
ready(cb: (err: Error | null) => void): Promise<this>;
```

Returns a Promise that resolves (to the queue) when the queue is connected to Redis and Lua scripts are cached. If the queue is already ready, resolves immediately.

```js
await queue.ready();
```

---

### `Queue#isRunning()`

```ts
isRunning(): boolean;
```

Returns `true` unless the queue is shutting down due to a call to `Queue#close()`.

---

### `Queue#getJob(jobId, [cb])`

```ts
getJob(jobId: string, cb: (job: Job<T>) => void): void;
getJob(jobId: string): Promise<Job<T>>;
```

Looks up a job by its ID. The returned job will emit events if `getEvents` and `storeJobs` are `true`.

**Warning:** Indiscriminate use can lead to increasing memory usage when `storeJobs` is `true`, as each queue maintains a Map of all associated jobs for event dispatch.

```js
const job = await queue.getJob('3');
console.log(job.status); // 'created' | 'succeeded' | 'failed' | 'retrying'
```

---

### `Queue#getJobs(type, page, [cb])`

```ts
getJobs(type: string, page: Page, cb: (jobs: Job<T>[]) => void): void;
getJobs(type: string, page: Page): Promise<Job<T>[]>;
```

Looks up jobs by queue type.

**Page parameter:**
```ts
interface Page {
  start?: number;  // For 'waiting', 'active', 'delayed' types
  end?: number;    // For 'waiting', 'active', 'delayed' types
  size?: number;   // For 'failed', 'succeeded' types
}
```

**Type values:** `'waiting'`, `'active'`, `'succeeded'`, `'failed'`, `'delayed'`

- For `waiting`, `active`, `delayed`: Use `{ start, end }` to specify index range.
- For `succeeded`, `failed`: Use `{ size }` to get an arbitrary subset (these are Redis SETs, unordered).

```js
// Get first 25 waiting jobs
const jobs = await queue.getJobs('waiting', { start: 0, end: 25 });

// Get up to 100 failed jobs
const failed = await queue.getJobs('failed', { size: 100 });
```

---

### `Queue#removeJob(jobId, [cb])`

```ts
removeJob(jobId: string): Promise<void>;
removeJob(jobId: string, cb: () => void): void;
```

Removes a job by its ID. **Idempotent.** Returns the Queue instance it was called on.

**Warning:** May have unintended side-effects if the job is currently being processed.

```js
await queue.removeJob('3');
```

---

### `Queue#saveAll(jobs)`

```ts
saveAll(jobs: Job<T>[]): Promise<Map<Job<T>, Error>>;
```

Bulk-saves an array of jobs using a pipelined Redis request. More efficient than saving individually when creating many jobs at once.

Returns a `Map<Job, Error>` -- jobs that saved successfully will NOT appear in the map. The map will often be empty.

Each job in the array is mutated with its assigned ID.

```js
const jobs = [
  queue.createJob({ x: 1 }),
  queue.createJob({ x: 2 }),
  queue.createJob({ x: 3 }),
];

const errors = await queue.saveAll(jobs);
// errors is a Map<Job, Error> - empty if all succeeded
// Each job now has its .id populated
```

---

### `Queue#destroy([cb])`

```ts
destroy(): Promise<void>;
destroy(cb: () => void): void;
```

Removes **all** Redis keys belonging to this queue. **Idempotent.** Returns the number of keys removed.

**Use with extreme care.**

```js
await queue.destroy();
```

---

## Queue Events (Local)

Local events are emitted on the queue instance that processes jobs. They fire on the worker that processed the job.

### `ready`

```js
queue.on('ready', () => { });
```

The queue has connected to Redis and Lua scripts are cached. Prefer using `Queue#ready()` Promise instead.

### `error`

```js
queue.on('error', (err: Error) => { });
```

Redis errors re-emitted from the queue. NOT emitted for failed jobs.

### `succeeded`

```js
queue.on('succeeded', (job: Job, result: any) => { });
```

This queue instance successfully processed the job. `result` is what the handler returned/resolved.

### `retrying`

```js
queue.on('retrying', (job: Job, err: Error) => { });
```

This queue instance processed the job but it failed and is being re-enqueued for another attempt. `job.options.retries` has been decremented. Stack trace added to `job.options.stacktraces` array.

### `failed`

```js
queue.on('failed', (job: Job, err: Error) => { });
```

This queue instance processed the job and it failed with no retries remaining.

### `stalled`

```js
queue.on('stalled', (jobId: string) => { });
```

A stalled job was detected. The emitting queue instance is the one that **detected** the stall, not necessarily the one that was processing the job.

---

## Queue Events (PubSub)

PubSub events are sent by workers (with `sendEvents: true`) and received by any queue listening (with `getEvents: true`). These are cross-process/cross-server events for monitoring all activity on the queue.

These events pass `jobId` (string), not a `Job` object, since the job may have been created in a different process.

### `job succeeded`

```js
queue.on('job succeeded', (jobId: string, result: any) => { });
```

### `job retrying`

```js
queue.on('job retrying', (jobId: string, err: Error) => { });
```

### `job failed`

```js
queue.on('job failed', (jobId: string, err: Error) => { });
```

### `job progress`

```js
queue.on('job progress', (jobId: string, progress: any) => { });
```

---

## Job Properties

| Property | Type | Description |
|----------|------|-------------|
| `id` | `string` | Job ID, unique per queue. Not populated until `.save()` resolves. Can be overridden with `Job#setId()`. |
| `data` | `T` (generic) | User data associated with the job. Must be JSON-serializable. Should be small (ideally < 1kB, never > 100kB). |
| `options` | `object` (readonly) | Internal options object. Contains `retries`, `backoff`, `timeout`, `delay`, `stacktraces`. Do not modify directly. |
| `queue` | `Queue` | Reference to the Queue that created, fetched, or is processing this job. |
| `progress` | `any` | Progress value, as reported by `reportProgress`. Not persisted to Redis. |
| `status` | `'created' \| 'succeeded' \| 'failed' \| 'retrying'` | Current status of the job. |

### Job `options` Internal Structure

```js
job.options = {
  timestamp: Number,       // Creation timestamp
  stacktraces: [],         // Array of error stack traces from failed attempts
  retries: 0,              // Remaining retry count (decremented on each retry)
  backoff: {               // Only if backoff() was called
    strategy: 'immediate', // 'immediate' | 'fixed' | 'exponential' | custom
    delay: 0               // Delay factor in ms
  },
  timeout: undefined,      // Timeout in ms (undefined = no timeout)
  delay: undefined         // delayUntil timestamp
};
```

---

## Job Methods (Chaining API)

All configuration methods return `this` for chaining (except `save` and `remove` which return Promises).

### `Job#setId(id)`

```ts
setId(id: string): this;
```

Explicitly sets the job ID. If a job with this ID already exists, the job will NOT be created and `job.id` will be set to `null`.

**Use case:** Idempotent job creation -- pass an external resource ID to ensure a job runs only once per resource.

**Warning:** Avoid passing numeric IDs, as they may conflict with auto-generated IDs.

When combined with `removeOnSuccess: true` and `removeOnFailure: true`, allows re-running the same job ID (effectively global concurrency of 1 for that ID).

```js
const job = queue.createJob({ userId: 'abc' })
  .setId('setup-user-abc')
  .save();
```

---

### `Job#retries(n)`

```ts
retries(n: number): this;
```

Sets the number of automatic retries on failure.

- Default: `0` (no retries).
- Stored in `job.options.retries` and decremented on each retry.

```js
queue.createJob(data).retries(3).save();
```

---

### `Job#backoff(strategy, [delayFactor])`

```ts
backoff(strategy: string, delayFactor?: number): this;
```

Sets the backoff policy for retries. Only meaningful when `retries > 0`.

**Built-in strategies:**

| Strategy | Behavior | `delayFactor` |
|----------|----------|---------------|
| `'immediate'` | Retry immediately (delay = 0) | Not used |
| `'fixed'` | Wait a fixed delay before each retry | ms to wait |
| `'exponential'` | Exponential backoff: delay doubles after each retry | Initial delay in ms |

Stored in `job.options.backoff` as `{ strategy, delay }`.

**Important:** `fixed` and `exponential` strategies require `activateDelayedJobs: true` on at least one Queue instance.

```js
queue.createJob(data).retries(3).backoff('immediate').save();
queue.createJob(data).retries(3).backoff('fixed', 1000).save();
queue.createJob(data).retries(3).backoff('exponential', 1000).save();
// exponential: 1s, 2s, 4s, 8s, ...
```

Default: `'immediate'`.

---

### `Job#delayUntil(dateOrTimestamp)`

```ts
delayUntil(dateOrTimestamp: Date | number): this;
```

Delay the job until the given Date or Unix timestamp (ms) passes.

Requires `activateDelayedJobs: true` on at least one Queue instance.

Default: immediate processing (no delay).

```js
queue.createJob(data)
  .delayUntil(Date.parse('2038-01-19T03:14:08.000Z'))
  .save();

queue.createJob(data)
  .delayUntil(Date.now() + 60000) // 1 minute from now
  .save();
```

---

### `Job#timeout(ms)`

```ts
timeout(milliseconds: number): this;
```

Sets a runtime timeout in milliseconds. If the handler takes longer than this to call `done` or resolve its Promise, the job is treated as failed (and retried if applicable).

Default: no timeout.

```js
queue.createJob(data).timeout(10000).save(); // 10 second timeout
```

---

### `Job#save([cb])`

```ts
save(): Promise<this>;
save(cb: (err: Error, job: this) => void): void;
```

Saves the job to Redis and enqueues it for processing. After resolution, `job.id` is populated.

Returns a Promise (in addition to calling the optional callback).

```js
const job = await queue.createJob(data).save();
console.log(job.id); // now populated
```

---

### `Job#reportProgress(data)`

```ts
reportProgress(p: any): void;
```

Reports progress from within a handler function. The argument can be **any JSON-serializable value** (not just a number).

- Causes a `progress` event on the job instance and a `job progress` PubSub event.
- Does NOT persist to Redis.
- Stored on `job.progress` in memory on all Queue instances that have `storeJobs` and `getEvents` enabled.

```js
queue.process(async (job) => {
  job.reportProgress(10);
  // ...
  job.reportProgress({ page: 3, totalPages: 11 });
  // ...
});
```

---

### `Job#remove([cb])`

```ts
remove(): Promise<this>;
remove(cb: (job: this) => void): void;
```

Removes the job from the queue. **Idempotent.**

Internally calls `Queue#removeJob(job.id)`. If you have the ID but not the job object, use `Queue#removeJob` directly for efficiency.

Returns the Job instance it was called on.

**Warning:** May have unintended side-effects if the job is currently being processed.

```js
await job.remove();
```

---

## Job Events

Job events are PubSub events emitted on the `Job` instance itself. Disabled when `getEvents` is `false`.

These events work cross-process -- the job instance receives events regardless of which worker processed it.

**Caveat:** Job events become unreliable across process restarts, since the queue loses its reference to the job object. Use Queue-level events for more reliability.

### `succeeded`

```js
job.on('succeeded', (result: any) => { });
```

The job completed successfully.

### `failed`

```js
job.on('failed', (err: Error) => { });
```

The job failed and is NOT being retried.

### `retrying`

```js
job.on('retrying', (err: Error) => { });
```

The job failed but is being automatically re-enqueued for another attempt.

### `progress`

```js
job.on('progress', (progress: any) => { });
```

The job handler called `reportProgress`.

---

## Health Check

### Return Shape

```ts
interface HealthCheckResult {
  waiting: number;    // Jobs waiting to be processed
  active: number;     // Jobs currently being processed
  succeeded: number;  // Jobs that completed successfully
  failed: number;     // Jobs that failed (exhausted retries)
  delayed: number;    // Jobs delayed until a future timestamp
  newestJob?: string; // ID of the newest job (if using default ID behavior)
}
```

Use `newestJob` to estimate job creation throughput over time. Combine with `waiting` and `active` counts to infer processing throughput.

```js
const health = await queue.checkHealth();
// {
//   waiting: 5,
//   active: 2,
//   succeeded: 1000,
//   failed: 12,
//   delayed: 3,
//   newestJob: '1015'
// }
```

---

## Backoff Strategies

### Built-in Strategies

| Name | Function | Description |
|------|----------|-------------|
| `'immediate'` | `() => 0` | Retry immediately with no delay. |
| `'fixed'` | `(job) => job.options.backoff.delay` | Wait a fixed delay (ms) before each retry. |
| `'exponential'` | `(job) => { delay = backoff.delay; backoff.delay *= 2; return delay; }` | Delay doubles after each retry. First retry uses the initial delay factor. |

### Custom Backoff Strategies

The `queue.backoffStrategies` Map can be extended with custom strategies:

```js
queue.backoffStrategies.set('linear', (job) => {
  return job.options.backoff.delay * (job.options.retries + 1);
});

// Then use it:
queue.createJob(data)
  .retries(5)
  .backoff('linear', 1000)
  .save();
```

The strategy function receives the `Job` object and must return a delay in milliseconds.

---

## Redis Connection Options

### Option Formats

The `redis` setting accepts three formats:

#### 1. Configuration Object (node_redis `ClientOpts`)

```js
{
  redis: {
    host: '127.0.0.1',
    port: 6379,
    db: 0,
    options: {},
    password: 'secret',
    socket: '/var/run/redis.sock', // Use instead of host/port
  }
}
```

If `socket` is provided, it is mapped to the `path` option internally.

#### 2. Redis URL String

```js
{
  redis: 'redis://user:password@host:port/db'
}
```

#### 3. Existing RedisClient Instance

```js
const redis = require('redis');
const client = redis.createClient(process.env.REDIS_URL);

{
  redis: client
}
```

When an existing client is passed:
- Bee-Queue issues normal commands over it.
- It calls `duplicate()` on the client for blocking commands (`brpoplpush`) and PubSub subscriptions if enabled.
- `quitCommandClient` defaults to `false` (won't quit the provided client on `Queue#close()`).

### Connection Count Per Queue Instance

Each Queue instance creates up to 3 Redis connections:

| Connection | Purpose | Created When |
|------------|---------|--------------|
| `client` | Normal Redis commands | Always |
| `bclient` | Blocking `brpoplpush` for job fetching | `isWorker: true` |
| `eclient` | PubSub subscription for events/delayed jobs | `getEvents: true` OR `activateDelayedJobs: true` |

### Connection Sharing Pattern

Producer queues (that only create jobs, not process them) can share a single Redis connection:

```js
const sharedConfig = {
  getEvents: false,
  isWorker: false,
  sendEvents: false,
  redis: redis.createClient(process.env.REDIS_URL),
};

const emailQueue = new Queue('email', sharedConfig);
const smsQueue = new Queue('sms', sharedConfig);
```

Worker queues will always `duplicate()` the provided client for blocking/PubSub, creating additional connections.

---

## Concurrency Model

- **Per-instance concurrency:** Set via `Queue#process(concurrency, handler)`.
- **Default concurrency:** `1` (one job at a time per Queue instance).
- **System-wide concurrency:** Total = `concurrency * number_of_worker_instances`.
- **Process can only call `.process()` once** per Queue instance.
- **Scale horizontally** by running more worker processes/servers, all connecting to the same Redis.
- Jobs are distributed via Redis `brpoplpush` (blocking pop), so idle workers receive jobs immediately upon enqueue (non-polling).
- The handler must never block the Node.js event loop for extended periods, or stall detection may trigger false positives.

---

## Stall Detection Internals

### Mechanism

Bee-Queue provides **at-least-once delivery** through stall detection.

1. Workers periodically "check in" to Redis for each active job (heartbeat).
2. The `stallInterval` setting controls the heartbeat window (default: 5000ms).
3. If a worker fails to check in within `stallInterval`, the job is considered stalled.
4. `checkStalledJobs` finds stalled jobs and re-enqueues them to the waiting list.

### Redis Implementation

Two Redis SETs are used for stall tracking:

- `bq:{name}:stalling` -- Snapshot of active job IDs at the start of the current stall interval.
- `bq:{name}:stallBlock` -- Used to prevent race conditions during stall checks.

**Flow:**
1. At the start of each stall interval, a snapshot of the active list is stored in the `stalling` set.
2. During the interval, workers remove their job IDs from the `stalling` set (checking in).
3. At the end of the interval, any IDs remaining in `stalling` are considered stalled.
4. Stalled jobs are moved back to the `waiting` list for re-processing.

### Best Practices

- `checkStalledJobs` is called **once automatically** when `process()` is first called.
- You should **also** call it manually with an `interval` parameter for ongoing checks.
- Recommended interval: same as or slightly shorter than `stallInterval`.
- Only 1-2 instances need to run the check (not all workers).
- Maximum delay from stall to retry: approximately `stallInterval + interval`.

```js
// Recommended: check every 5 seconds
queue.checkStalledJobs(5000);
```

---

## Redis Key Layout

All keys are prefixed with `{prefix}:{name}:` (default: `bq:{name}:`).

| Key | Type | Description |
|-----|------|-------------|
| `bq:{name}:id` | Integer | Auto-incrementing counter for Job IDs. |
| `bq:{name}:jobs` | Hash | Maps Job ID to JSON string of `{ data, options }`. |
| `bq:{name}:waiting` | List | IDs of jobs waiting to be processed. |
| `bq:{name}:active` | List | IDs of jobs currently being processed. |
| `bq:{name}:succeeded` | Set | IDs of jobs that succeeded. |
| `bq:{name}:failed` | Set | IDs of jobs that failed. |
| `bq:{name}:delayed` | Sorted Set | Maps delayed timestamps to Job IDs. |
| `bq:{name}:stalling` | Set | IDs of jobs that haven't checked in during the current interval. |
| `bq:{name}:stallBlock` | Set | IDs used to prevent stall detection race conditions. |
| `bq:{name}:events` | PubSub Channel | Channel for workers to publish job results/progress. |
| `bq:{name}:earlierDelayed` | PubSub Channel | Published when a new delayed job is added before all others. |

---

## Delayed Job Activation

- Controlled by `activateDelayedJobs` setting (default: `false`).
- Must be `true` on at least one Queue instance for delayed jobs and `fixed`/`exponential` backoff to work.
- `delayedDebounce` setting groups activations within a window to reduce churn.
- `nearTermWindow` setting determines the maximum polling interval for delayed job activation as a safety net.

**Internal flow:**
1. When a delayed job is saved, it goes into the `delayed` sorted set with its timestamp as the score.
2. The Queue with `activateDelayedJobs: true` monitors the `earlierDelayed` PubSub channel.
3. When a delayed job's timestamp passes, it is moved from `delayed` to `waiting`.
4. An `EagerTimer` internally manages `setTimeout` scheduling within the `nearTermWindow`.

---

## Bulk Operations

### `Queue#saveAll(jobs)`

The only bulk operation available. Uses Redis pipelining for network efficiency.

```js
const jobs = Array.from({ length: 1000 }, (_, i) =>
  queue.createJob({ index: i })
);

const errors = await queue.saveAll(jobs);
if (errors.size > 0) {
  for (const [job, error] of errors) {
    console.error(`Job failed to save:`, error);
  }
}
```

**Notes:**
- Jobs are mutated in-place with their assigned IDs.
- Returns `Map<Job, Error>` -- empty map means all succeeded.
- Much faster than individual `save()` calls for large batches.

---

## Graceful Shutdown Pattern

```js
const TIMEOUT = 30 * 1000;

async function shutdown() {
  try {
    await queue.close(TIMEOUT);
  } catch (err) {
    console.error('Failed to shut down gracefully', err);
  }
  process.exit(0);
}

process.on('SIGTERM', shutdown);
process.on('SIGINT', shutdown);

process.on('uncaughtException', async (err) => {
  console.error('Uncaught exception:', err);
  try {
    await queue.close(TIMEOUT);
  } catch (closeErr) {
    console.error('Failed to shut down gracefully', closeErr);
  }
  process.exit(1);
});
```

- `Queue#close()` is **idempotent** -- safe to call multiple times.
- Active jobs that don't finish within `timeout` will stall and be retried by another worker.
- Make jobs **idempotent** since at-least-once delivery means jobs may be processed more than once.

---

## TypeScript Types

```ts
import BeeQueue from 'bee-queue';

// Queue with typed job data
const queue = new BeeQueue<{ x: number; y: number }>('math');

// Full type definitions
declare class BeeQueue<T = any> extends EventEmitter {
  name: string;
  keyPrefix: string;
  jobs: Map<string, BeeQueue.Job<T>>;
  paused: boolean;
  settings: any;
  backoffStrategies: Map<string, (job: BeeQueue.Job<T>) => number>;

  constructor(name: string, settings?: BeeQueue.QueueSettings);

  // Event overloads
  on(ev: 'ready', fn: () => void): this;
  on(ev: 'error', fn: (err: Error) => void): this;
  on(ev: 'succeeded', fn: (job: BeeQueue.Job<T>, result: any) => void): this;
  on(ev: 'retrying', fn: (job: BeeQueue.Job<T>, err: Error) => void): this;
  on(ev: 'failed', fn: (job: BeeQueue.Job<T>, err: Error) => void): this;
  on(ev: 'stalled', fn: (jobId: string) => void): this;
  on(ev: 'job succeeded', fn: (jobId: string, result: any) => void): this;
  on(ev: 'job retrying', fn: (jobId: string, err: Error) => void): this;
  on(ev: 'job failed', fn: (jobId: string, err: Error) => void): this;
  on(ev: 'job progress', fn: (jobId: string, progress: any) => void): this;

  // Methods
  ready(): Promise<this>;
  isRunning(): boolean;
  connect(): Promise<boolean>;
  createJob<U extends T>(data: U): BeeQueue.Job<U>;
  getJob(jobId: string): Promise<BeeQueue.Job<T>>;
  getJobs(type: string, page: BeeQueue.Page): Promise<BeeQueue.Job<T>[]>;
  process<U>(handler: (job: BeeQueue.Job<T>) => Promise<U>): void;
  process<U>(concurrency: number, handler: (job: BeeQueue.Job<T>) => Promise<U>): void;
  process<U>(handler: (job: BeeQueue.Job<T>, done: BeeQueue.DoneCallback<U>) => void): void;
  process<U>(concurrency: number, handler: (job: BeeQueue.Job<T>, done: BeeQueue.DoneCallback<U>) => void): void;
  checkStalledJobs(interval?: number): Promise<number>;
  checkHealth(): Promise<BeeQueue.HealthCheckResult>;
  close(timeout?: number | null): Promise<void>;
  removeJob(jobId: string): Promise<void>;
  destroy(): Promise<void>;
  saveAll(jobs: BeeQueue.Job<T>[]): Promise<Map<BeeQueue.Job<T>, Error>>;
}

declare namespace BeeQueue {
  interface QueueSettings {
    prefix?: string;
    stallInterval?: number;
    nearTermWindow?: number;
    delayedDebounce?: number;
    redis?: ClientOpts | RedisClient | string;
    isWorker?: boolean;
    getEvents?: boolean;
    sendEvents?: boolean;
    storeJobs?: boolean;
    ensureScripts?: boolean;
    activateDelayedJobs?: boolean;
    removeOnSuccess?: boolean;
    removeOnFailure?: boolean;
    quitCommandClient?: boolean;
    redisScanCount?: number;
    autoConnect?: boolean;
  }

  interface Job<T> extends EventEmitter {
    id: string;
    data: T;
    readonly options: any;
    queue: BeeQueue<T>;
    progress: any;
    status: 'created' | 'succeeded' | 'failed' | 'retrying';

    on(ev: 'succeeded', fn: (result: any) => void): this;
    on(ev: 'retrying', fn: (err: Error) => void): this;
    on(ev: 'failed', fn: (err: Error) => void): this;
    on(ev: 'progress', fn: (progress: any) => void): this;

    setId(id: string): this;
    retries(n: number): this;
    backoff(strategy: string, delayFactor?: number): this;
    delayUntil(dateOrTimestamp: Date | number): this;
    timeout(milliseconds: number): this;
    save(): Promise<this>;
    reportProgress(p: any): void;
    remove(): Promise<this>;
  }

  interface Page {
    start?: number;
    end?: number;
    size?: number;
  }

  interface HealthCheckResult {
    waiting: number;
    active: number;
    succeeded: number;
    failed: number;
    delayed: number;
    newestJob?: string;
  }

  type DoneCallback<T> = (error: Error | null, result?: T) => void;
}
```

---

## Features NOT Supported by Bee Queue

These are explicitly out of scope (relevant for migration planning):

- **Job priority** -- No built-in priority. Workaround: use multiple queues.
- **Repeatable/recurring jobs** -- Not supported. Must be implemented externally (e.g., cron).
- **Worker tracking** -- No built-in worker registry.
- **All-workers pause/resume** -- No global pause mechanism.
- **Rate limiting** -- Not supported.
- **Job dependencies/flows** -- Not supported.
- **Named processors** -- Only one `process()` call per queue instance.
- **Global events beyond PubSub** -- No Redis Streams support.
- **Job data persistence after completion** -- Relies on `removeOnSuccess`/`removeOnFailure` settings; no TTL-based expiry.

---

## Web Interface

[Arena](https://github.com/bee-queue/arena) -- Web UI for managing Bee-Queue jobs and inspecting queue health. Supports both Bee-Queue and Bull.

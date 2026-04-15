# Learning Guide: Building a RESP-Compatible Message Queue Server in Rust

**Generated**: 2026-04-03
**Sources**: 42 resources analyzed
**Depth**: deep

## Prerequisites

- Proficiency in Rust (ownership, traits, async/await, tokio)
- Understanding of TCP networking and binary protocols
- Familiarity with Redis concepts (commands, data types, pipelining)
- Familiarity with at least one MQ library (BullMQ, Sidekiq, Celery, or glide-mq)
- Basic understanding of Lua scripting

## TL;DR

- RESP is a simple, CRLF-terminated, prefix-based wire protocol. Implementing a server requires parsing 5 RESP2 types (simple strings, errors, integers, bulk strings, arrays) plus optionally 8 RESP3 types.
- BullMQ is the hardest client to support - it ships 51 Lua scripts using EVAL/EVALSHA with up to 14 keys per script. Sidekiq and Celery are simpler (no Lua, pure commands).
- Use `mlua` with Lua 5.1/LuaJIT to embed Lua. Implement `redis.call()`/`redis.pcall()` as Rust functions registered into the Lua VM. Cache scripts by SHA1 hash.
- For queue-optimized storage, use skip lists for sorted sets (delayed/priority jobs), arena allocators for job payloads, and a sharded DashMap for the keyspace.
- A shared-nothing, thread-per-core architecture (like DragonflyDB) outperforms single-threaded Redis by 25x. Start simpler with Tokio multi-threaded runtime and shard data by key hash.

## 1. RESP Protocol Implementation

### Wire Format

RESP (Redis Serialization Protocol) is a request-response protocol over TCP (default port 6379). Every element is terminated with `\r\n` (CRLF). The first byte identifies the type.

**RESP2 Types (must implement)**:

| First Byte | Type | Example |
|-----------|------|---------|
| `+` | Simple String | `+OK\r\n` |
| `-` | Error | `-ERR unknown command\r\n` |
| `:` | Integer (signed 64-bit) | `:1000\r\n` |
| `$` | Bulk String (length-prefixed) | `$5\r\nhello\r\n` |
| `*` | Array (count-prefixed) | `*2\r\n$3\r\nGET\r\n$3\r\nfoo\r\n` |

Null values in RESP2: `$-1\r\n` (null bulk string) or `*-1\r\n` (null array).

**RESP3 Types (implement for modern clients)**:

| First Byte | Type | Example |
|-----------|------|---------|
| `_` | Null | `_\r\n` |
| `#` | Boolean | `#t\r\n` |
| `,` | Double | `,1.23\r\n` |
| `%` | Map | `%2\r\n+key\r\n:1\r\n+key2\r\n:2\r\n` |
| `~` | Set | `~2\r\n+a\r\n+b\r\n` |
| `>` | Push (out-of-band) | Server-initiated messages |
| `(` | Big Number | `(3492890328409238509324850943850943825024385\r\n` |
| `=` | Verbatim String | `=15\r\ntxt:Some string\r\n` |

**Client requests** are always arrays of bulk strings:
```
*3\r\n$3\r\nSET\r\n$3\r\nfoo\r\n$3\r\nbar\r\n
```

**Protocol negotiation**: Clients send `HELLO 3` to upgrade to RESP3. Respond with a map containing `server`, `version`, `proto`, `id`, `mode`, `role` fields. If you only support RESP2, respond with `-NOPROTO` error.

**Inline commands**: For telnet debugging, support space-separated commands without RESP framing (detect by absence of `*` as first byte).

**Pipelining**: Clients can send multiple commands without waiting for responses. Responses must be in the same order as requests - this is purely a buffering concern, not a special protocol mode.

### Rust Implementation Strategy

**Option A: Use `redis-protocol` crate** - provides zero-copy RESP2/RESP3 parsing with three frame interfaces:
- `OwnedFrame` - simplest, allocates per frame
- `BytesFrame` - zero-copy with `bytes::Bytes`
- `RangeFrame` - lowest overhead, borrowed references

**Option B: Hand-roll (recommended for MQ server)** - mini-redis demonstrates the pattern:

```rust
pub enum Frame {
    Simple(String),
    Error(String),
    Integer(i64),
    Bulk(Bytes),
    Null,
    Array(Vec<Frame>),
}
```

**Parse-or-read-more loop** (from mini-redis):
1. Attempt to parse a complete frame from the buffer
2. If `Incomplete`, read more bytes from TCP into `BytesMut`
3. If complete, advance the buffer cursor and return the frame
4. If TCP returns 0 bytes, the connection is closed

```rust
// Pseudocode for the core read loop
pub async fn read_frame(&mut self) -> Result<Option<Frame>> {
    loop {
        if let Some(frame) = self.parse_frame()? {
            return Ok(Some(frame));
        }
        if 0 == self.stream.read_buf(&mut self.buffer).await? {
            return Ok(None); // Connection closed
        }
    }
}
```

**Key optimization**: Use `Cursor<&[u8]>` for parsing. Check completeness first (`Frame::check`) without allocating, then parse. This avoids partial allocations on incomplete reads.

### TLS Support

Use `tokio-rustls` with `TlsAcceptor`:
1. Load certificates via `rustls::ServerConfig`
2. Wrap each accepted `TcpStream` with `acceptor.accept(stream).await`
3. The resulting `TlsStream` implements `AsyncRead + AsyncWrite` - the rest of the code is unchanged

For per-connection certificate selection (e.g., SNI-based), use `LazyConfigAcceptor`.

### AUTH and Connection Commands

Implement these connection-management commands:

| Command | Behavior |
|---------|----------|
| `PING` | Return `+PONG\r\n` |
| `AUTH password` | Validate password, set connection authenticated flag |
| `HELLO [proto] [AUTH user pass]` | Protocol negotiation, optional inline auth |
| `CLIENT SETNAME name` | Store connection name for debugging |
| `CLIENT GETNAME` | Return stored name |
| `SELECT db` | For MQ, accept but ignore (or support 0-15 databases) |
| `QUIT` | Close connection gracefully |
| `INFO [section]` | Return server info (clients need this for health checks) |
| `CONFIG GET param` | Return dummy values for expected params |

## 2. Redis Command Subset for MQ Workloads

### BullMQ Command Surface (Most Complex)

BullMQ ships 51 Lua scripts. The Redis commands called across all scripts:

**Core Data Operations**:
- `RPOPLPUSH` / `LMOVE` - atomically move jobs between lists (wait -> active)
- `RPUSH` / `LPUSH` - add jobs to lists (FIFO/LIFO)
- `LREM` - remove specific items from lists
- `LLEN` - list length
- `ZADD` - add to sorted sets (delayed jobs, completed/failed with timestamps)
- `ZRANGEBYSCORE` / `ZRANGE` - query delayed jobs ready for execution
- `ZREM` - remove from sorted sets
- `ZCARD` / `SCARD` - count members
- `HSET` / `HMSET` / `HGET` / `HMGET` / `HGETALL` / `HDEL` - job data storage
- `INCR` - job ID generation
- `EXISTS` - key existence checks
- `SADD` / `SREM` / `SMEMBERS` / `SISMEMBER` - set operations for dependencies
- `XADD` - event stream emission
- `XREADGROUP` / `XACK` - consumer group reads (for QueueEvents)
- `XGROUP CREATE` - create consumer groups
- `EVAL` / `EVALSHA` - Lua script execution
- `SCRIPT LOAD` / `SCRIPT EXISTS` - script caching

**Key Patterns** (prefix is configurable, default `bull`):
- `bull:{queueName}:wait` - waiting jobs list
- `bull:{queueName}:active` - processing jobs list
- `bull:{queueName}:delayed` - delayed jobs sorted set
- `bull:{queueName}:completed` - completed jobs sorted set
- `bull:{queueName}:failed` - failed jobs sorted set
- `bull:{queueName}:prioritized` - priority queue sorted set
- `bull:{queueName}:events` - event stream
- `bull:{queueName}:id` - job ID counter
- `bull:{queueName}:meta` - queue metadata hash
- `bull:{queueName}:{jobId}` - individual job hash

### Sidekiq Command Surface (Simpler)

Sidekiq uses no Lua scripts - pure Redis commands with pipelining:

**Enqueue**: `LPUSH queue:{name} {json}` + `SADD queues {name}`
**Schedule**: `ZADD schedule {timestamp} {json}`
**Consume**: `BRPOP queue:{name1} queue:{name2} ... {timeout}`
**Retry**: `ZADD retry {timestamp} {json}`
**Track**: `SADD queues {name}` (queue discovery)

**Key Patterns**:
- `queue:{name}` - job lists
- `schedule` - sorted set of scheduled jobs
- `retry` - sorted set of jobs to retry
- `queues` - set of all queue names

### Celery Command Surface

Celery (via Kombu transport) also uses no Lua scripts:

**Enqueue**: `LPUSH {queue_name} {message}`
**Consume**: `BRPOP {queue1} {queue2} ... {timeout}`
**Results**: `SET celery-task-meta-{task_id} {result}` + `PUBLISH {task_id} {result}`
**Priority**: Separate keys per priority level: `{queue}\x06\x16{priority}`
**Unacked**: `ZADD unacked {timestamp} {message}` (visibility timeout tracking)
**Fanout**: `PUBLISH` / `PSUBSCRIBE` for fanout exchanges

### Minimum Command Set

Commands required to support all four client libraries:

**Must Implement (Full)**:
- String: `SET`, `GET`, `DEL`, `EXISTS`, `INCR`, `INCRBY`, `EXPIRE`, `PEXPIRE`, `TTL`, `PTTL`, `SETEX`, `MGET`, `PERSIST`
- List: `LPUSH`, `RPUSH`, `LPOP`, `RPOP`, `BRPOP`, `BLPOP`, `LREM`, `LLEN`, `LRANGE`, `LINDEX`, `RPOPLPUSH`/`LMOVE`
- Hash: `HSET`, `HGET`, `HMSET`, `HMGET`, `HGETALL`, `HDEL`, `HINCRBY`, `HSETNX`, `HEXISTS`
- Set: `SADD`, `SREM`, `SMEMBERS`, `SISMEMBER`, `SCARD`
- Sorted Set: `ZADD`, `ZREM`, `ZRANGEBYSCORE`, `ZRANGE`, `ZREVRANGEBYSCORE`, `ZCARD`, `ZSCORE`, `ZREVRANGE`
- Stream: `XADD`, `XREADGROUP`, `XACK`, `XGROUP CREATE`, `XLEN`, `XRANGE`, `XDEL`, `XTRIM`, `XINFO`
- Scripting: `EVAL`, `EVALSHA`, `SCRIPT LOAD`, `SCRIPT EXISTS`, `SCRIPT FLUSH`
- Transaction: `MULTI`, `EXEC`, `DISCARD`, `WATCH`, `UNWATCH`
- Pub/Sub: `SUBSCRIBE`, `UNSUBSCRIBE`, `PSUBSCRIBE`, `PUNSUBSCRIBE`, `PUBLISH`
- Key: `DEL`, `EXISTS`, `EXPIRE`, `PEXPIRE`, `TTL`, `PTTL`, `TYPE`, `KEYS`, `SCAN`, `RENAME`

**Stub/No-Op (return sensible defaults)**:
- `PING` - return PONG
- `SELECT` - accept, track per-connection DB index
- `INFO` - return minimal server info
- `CONFIG GET` - return expected defaults (e.g., `maxmemory-policy noeviction`)
- `CLIENT SETNAME` / `CLIENT GETNAME` / `CLIENT ID` - connection metadata
- `DBSIZE` - return key count
- `FLUSHDB` / `FLUSHALL` - clear data (implement fully or no-op with warning)
- `COMMAND` / `COMMAND DOCS` / `COMMAND COUNT` - command metadata
- `TIME` - return server time
- `WAIT` - return immediately (no replication)

## 3. Lua Script Embedding

### Why Lua Is Required

BullMQ is the primary driver for Lua support. It ships 51 Lua scripts that must execute atomically. These scripts are the core of BullMQ's reliability guarantees - they ensure that multi-step queue operations (like moving a job from waiting to active while checking rate limits and priorities) happen without race conditions.

Sidekiq and Celery do not use Lua scripts and can be supported without Lua. However, supporting Lua is essential for BullMQ (and many other Redis-based libraries that rely on atomic scripting).

### Choosing a Lua Crate

| Crate | Lua Version | Safety | Performance | Maturity | Recommendation |
|-------|-------------|--------|-------------|----------|----------------|
| **mlua** | 5.1-5.5, LuaJIT, Luau | Safe (IntoLua/FromLua) | Excellent with LuaJIT | Production | **Use this** |
| rlua | 5.3 | Safe | Good | Maintenance mode | Avoid |
| piccolo | ~5.3 | Pure Rust | Unknown | Experimental (v0.3) | Not ready |

**Use mlua with Lua 5.1 feature** - Redis uses Lua 5.1, and BullMQ scripts are written against Lua 5.1 semantics. Enable LuaJIT for performance when available.

### Implementation Architecture

```rust
use mlua::prelude::*;

struct LuaEngine {
    /// Script cache: SHA1 hex -> compiled Lua chunk
    scripts: DashMap<String, Vec<u8>>,
    /// Pool of Lua VMs (one per worker thread)
    vms: Vec<Lua>,
}

impl LuaEngine {
    fn new(num_threads: usize) -> Self {
        let vms: Vec<Lua> = (0..num_threads)
            .map(|_| {
                let lua = Lua::new_with(
                    StdLib::TABLE | StdLib::STRING | StdLib::MATH | StdLib::BIT,
                    LuaOptions::default(),
                ).unwrap();
                // Register redis.call() and redis.pcall()
                Self::register_redis_api(&lua);
                lua
            })
            .collect();
        Self { scripts: DashMap::new(), vms }
    }
}
```

### Implementing redis.call() and redis.pcall()

The key challenge: Lua scripts call `redis.call("SET", key, value)` which must execute against your server's data structures, not a real Redis.

```rust
fn register_redis_api(lua: &Lua) {
    let redis_table = lua.create_table().unwrap();

    // redis.call() - raises Lua error on Redis error
    let call_fn = lua.create_function(|lua, args: LuaMultiValue| {
        let cmd = extract_command(args)?;
        match execute_command(&cmd) {
            Ok(result) => redis_to_lua(lua, result),
            Err(e) => Err(LuaError::RuntimeError(e.to_string())),
        }
    }).unwrap();

    // redis.pcall() - returns error as table
    let pcall_fn = lua.create_function(|lua, args: LuaMultiValue| {
        let cmd = extract_command(args)?;
        match execute_command(&cmd) {
            Ok(result) => redis_to_lua(lua, result),
            Err(e) => {
                let err_table = lua.create_table()?;
                err_table.set("err", e.to_string())?;
                Ok(LuaValue::Table(err_table))
            }
        }
    }).unwrap();

    redis_table.set("call", call_fn).unwrap();
    redis_table.set("pcall", pcall_fn).unwrap();
    lua.globals().set("redis", redis_table).unwrap();
}
```

### Type Conversion (Redis <-> Lua)

Follow the official Redis-Lua type mapping:

| Redis Type | Lua Type | Notes |
|-----------|----------|-------|
| Integer | number | |
| Bulk String | string | |
| Array | table (1-indexed) | |
| Status Reply | table with `ok` field | `{ok = "OK"}` |
| Error Reply | table with `err` field | `{err = "ERR msg"}` |
| Nil | `false` | Critical - nil becomes false, not nil |

The reverse (Lua -> Redis):

| Lua Type | Redis Type | Notes |
|----------|-----------|-------|
| number | Integer | Truncated to i64 |
| string | Bulk String | |
| table (array) | Array | |
| table with `ok` | Status Reply | |
| table with `err` | Error Reply | |
| `false` / `nil` | Null | |
| boolean `true` | Integer 1 | |

### EVAL/EVALSHA Flow

```
EVAL <script> <numkeys> <key1> ... <keyN> <arg1> ... <argM>
```

1. Compute SHA1 of script body
2. Store in script cache: `scripts[sha1_hex] = compiled_chunk`
3. Set `KEYS` and `ARGV` arrays in Lua globals
4. Execute the chunk
5. Convert return value to RESP

```
EVALSHA <sha1> <numkeys> <key1> ... <keyN> <arg1> ... <argM>
```

1. Look up SHA1 in script cache
2. If not found, return `-NOSCRIPT No matching script`
3. If found, execute as above

### Script Pre-Loading Optimization

BullMQ always calls `EVALSHA` first, then falls back to `EVAL` on `NOSCRIPT`. You can optimize by:

1. **Pre-loading known scripts**: At startup, parse BullMQ's 51 Lua scripts from a bundled directory and pre-populate the script cache
2. **Script fingerprinting**: When you see the first EVAL from a connection, hash and cache it. Subsequent EVALSHA calls hit the cache
3. **Native fast-paths**: For the most critical scripts (addStandardJob, moveToActive, moveToFinished), implement the logic directly in Rust and detect the script by SHA1 hash. Fall back to Lua execution for unknown scripts

### Atomicity Guarantees

During Lua script execution:
- No other commands are processed on the same data shard
- All `redis.call()` invocations execute against the same consistent snapshot
- If the script fails midway, partial changes are NOT rolled back (matching Redis behavior)

### Additional Lua APIs to Implement

- `redis.log(level, message)` - log from within scripts
- `redis.sha1hex(string)` - compute SHA1
- `redis.error_reply(msg)` - construct error return
- `redis.status_reply(msg)` - construct status return
- `cjson.encode()` / `cjson.decode()` - JSON support (BullMQ uses this)
- `cmsgpack.pack()` / `cmsgpack.unpack()` - MessagePack support

For cjson, either compile Lua cjson as a C library loaded into mlua, or implement JSON encode/decode as Rust functions registered into the Lua VM.

## 4. Queue-Optimized Data Structures

### Keyspace Architecture

Unlike Redis (which is a general-purpose key-value store), your server can optimize data structures specifically for queue patterns:

```rust
/// The main keyspace - sharded for concurrency
struct Keyspace {
    /// N shards, accessed via hash(key) % N
    shards: Vec<Shard>,
}

struct Shard {
    /// Key -> Value store
    data: parking_lot::RwLock<HashMap<Bytes, Value>>,
    /// Key -> Expiry time
    expires: parking_lot::Mutex<BTreeMap<Instant, Vec<Bytes>>>,
    /// Blocking operations waiting on keys
    blocking: parking_lot::Mutex<HashMap<Bytes, Vec<BlockingWaiter>>>,
}

enum Value {
    String(Bytes),
    List(VecDeque<Bytes>),
    Hash(HashMap<Bytes, Bytes>),
    Set(HashSet<Bytes>),
    SortedSet(SortedSetImpl),
    Stream(StreamImpl),
}
```

### Sorted Sets (Critical for Delayed Jobs)

Redis sorted sets are the backbone of delayed job scheduling. Every MQ library uses ZADD/ZRANGEBYSCORE to manage delayed and scheduled jobs.

**Option A: Skip List** (like Redis)
- Use `crossbeam-skiplist::SkipMap<(OrderedFloat<f64>, Bytes), ()>` for concurrent access
- Good for high write frequency
- Lock-free reads
- O(log n) insert/remove/range queries

**Option B: BTreeMap with RwLock** (simpler, often faster for reads)
- `RwLock<BTreeMap<OrderedFloat<f64>, Vec<Bytes>>>` per shard
- Faster range queries than skip lists
- Lock contention under heavy writes

**Recommendation**: Start with `BTreeMap` inside `parking_lot::RwLock` per shard. The shard-level locking already reduces contention. Only switch to skip lists if profiling shows lock contention is the bottleneck.

### Lists (Job Queues)

RPOPLPUSH (wait -> active) is the most performance-critical operation:

```rust
/// Optimized for head/tail operations
struct QueueList {
    /// VecDeque for O(1) push/pop at both ends
    data: VecDeque<Bytes>,
}
```

For `LREM` (used by BullMQ to clean up active lists), maintain a secondary `HashMap<Bytes, usize>` counting occurrences to enable O(1) existence checks before O(n) removal.

### Time Wheel for Delayed Jobs

For the delayed job scheduler (promoting jobs from `delayed` sorted set to `wait` list when their timestamp arrives), a hierarchical time wheel is more efficient than polling sorted sets:

```rust
struct TimeWheel {
    /// Slots for the current second (1000 ms-resolution slots)
    current: Vec<Vec<JobId>>,
    /// Coarse slots for the current minute (60 slots)
    minutes: Vec<Vec<JobId>>,
    /// Background sorted set for far-future jobs (>1 hour)
    far_future: BTreeMap<Instant, Vec<JobId>>,
    /// Current position
    current_tick: u64,
}
```

Benefits over polling ZRANGEBYSCORE:
- O(1) to check if any jobs are due (just check current slot)
- O(k) to promote k due jobs (no scanning)
- Still falls back to sorted set for far-future scheduling

### Arena Allocation for Job Payloads

Job payloads are typically written once and read a few times during processing. Use `bumpalo` for batch allocation:

```rust
/// Per-request arena for temporary allocations
struct RequestArena {
    bump: bumpalo::Bump,
}

impl RequestArena {
    fn alloc_response(&self, data: &[u8]) -> &[u8] {
        self.bump.alloc_slice_copy(data)
    }

    fn reset(&mut self) {
        self.bump.reset();
    }
}
```

For long-lived job data, use standard allocation with `Bytes` for zero-copy sharing.

### Hash Optimization for Job Metadata

BullMQ stores each job as a Redis hash with fields like `data`, `opts`, `timestamp`, `returnvalue`, `stacktrace`, etc. Optimize for the common case:

```rust
/// Small hashes use a flat Vec (like Redis listpack)
/// Large hashes use HashMap
enum HashValue {
    Small(Vec<(Bytes, Bytes)>),    // <= 64 entries
    Large(HashMap<Bytes, Bytes>),   // > 64 entries
}
```

## 5. Server Architecture

### Recommended Architecture: Sharded Tokio

```
                    ┌─────────────────┐
                    │   TCP Listener   │
                    │  (TlsAcceptor)   │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  Accept Loop     │
                    │  (Semaphore)     │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
        ┌─────▼─────┐ ┌─────▼─────┐ ┌─────▼─────┐
        │ Connection │ │ Connection │ │ Connection │
        │   Task 1   │ │   Task 2   │ │   Task N   │
        └─────┬─────┘ └─────┬─────┘ └─────┬─────┘
              │              │              │
        ┌─────▼──────────────▼──────────────▼─────┐
        │           Command Router                 │
        │   (hash key -> shard, dispatch)          │
        └─────┬──────────────┬──────────────┬─────┘
              │              │              │
        ┌─────▼─────┐ ┌─────▼─────┐ ┌─────▼─────┐
        │  Shard 0   │ │  Shard 1   │ │  Shard N   │
        │  (data +   │ │  (data +   │ │  (data +   │
        │   Lua VM)  │ │   Lua VM)  │ │   Lua VM)  │
        └───────────┘ └───────────┘ └───────────┘
```

### Connection Handling

```rust
async fn run_server(addr: &str) -> Result<()> {
    let listener = TcpListener::bind(addr).await?;
    let semaphore = Arc::new(Semaphore::new(10_000)); // max connections
    let state = Arc::new(ServerState::new(num_cpus::get()));

    loop {
        let permit = semaphore.clone().acquire_owned().await?;
        let (socket, peer) = listener.accept().await?;
        let state = state.clone();

        tokio::spawn(async move {
            let mut conn = Connection::new(socket);
            let mut ctx = ConnectionContext::new(peer);

            while let Some(frame) = conn.read_frame().await? {
                let cmd = Command::from_frame(frame)?;
                let response = state.execute(&mut ctx, cmd).await;
                conn.write_frame(&response).await?;
            }

            drop(permit);
            Ok::<_, Error>(())
        });
    }
}
```

### Shared State Patterns

**For the keyspace**: Use sharding with `parking_lot::RwLock`:

```rust
struct ServerState {
    shards: Vec<Shard>,
    num_shards: usize,
    lua_engine: LuaEngine,
    script_cache: DashMap<String, Vec<u8>>,
    pubsub: PubSubManager,
}

impl ServerState {
    fn shard_for_key(&self, key: &[u8]) -> &Shard {
        let hash = crc16_xmodem(key);
        &self.shards[(hash as usize) % self.num_shards]
    }
}
```

**Number of shards**: Use `num_cpus::get() * 4` for good distribution without excessive overhead.

**For blocking operations** (BRPOP, BLPOP): Use `tokio::sync::Notify` or `broadcast` channels. When a LPUSH arrives on a key that has waiting BRPOP clients, wake the oldest waiter.

### Multi-Threading Models (Lessons from Existing Projects)

**DragonflyDB approach** (shared-nothing):
- One proactor thread per CPU core
- Each thread owns its data shard exclusively
- Commands on a single shard execute without locks
- Multi-key commands spanning shards use brief cross-shard coordination
- 25x throughput over single-threaded Redis

**KeyDB approach** (shared-state):
- Multiple I/O threads sharing the same data
- Global lock with fine-grained locking where possible
- Active replication for HA

**Recommendation for MQ server**: Start with Tokio multi-threaded runtime with sharded state. This gives you 80% of the shared-nothing benefits with much simpler code. For multi-key Lua scripts, route to the shard of KEYS[1] and access other shards via cross-shard calls.

### io_uring Consideration

`tokio-uring` and `glommio` offer io_uring-based I/O on Linux 5.8+:

**Pros**: Lower syscall overhead, kernel-side batching, potentially lower latency
**Cons**: Linux-only, immature ecosystem, ownership-based buffer model incompatible with standard async traits, single-threaded executors

**Recommendation**: Start with standard Tokio (`epoll` on Linux, `kqueue` on macOS). Add io_uring as an optional feature behind a `--io-uring` flag for Linux deployments. Glommio's thread-per-core model is architecturally interesting but locks you into Linux.

## 6. Persistence and Durability

### When Persistence Matters for MQ

| Scenario | Persistence Needed | Strategy |
|----------|-------------------|----------|
| Ephemeral task processing | No | In-memory only |
| Job durability required | Yes | AOF with everysec fsync |
| Must survive full restart | Yes | AOF + periodic snapshots |
| Audit trail needed | Yes | AOF without rewrite |
| High-throughput, some loss OK | Maybe | Periodic snapshots only |

### AOF Implementation for Queue Server

Simpler than Redis since you only need to log queue-mutation commands:

```rust
struct WriteAheadLog {
    file: tokio::fs::File,
    buffer: BytesMut,
    fsync_policy: FsyncPolicy,
    /// Commands since last fsync
    pending_count: u64,
}

enum FsyncPolicy {
    Always,       // fsync after every write (safest, slowest)
    EverySec,     // fsync once per second (good balance)
    No,           // let OS decide (fastest, risk data loss)
}

impl WriteAheadLog {
    async fn append(&mut self, cmd: &Command) -> Result<()> {
        // Write RESP-encoded command to buffer
        cmd.encode_resp(&mut self.buffer);
        self.file.write_all(&self.buffer).await?;
        self.buffer.clear();
        self.pending_count += 1;

        if self.fsync_policy == FsyncPolicy::Always {
            self.file.sync_all().await?;
        }
        Ok(())
    }
}
```

**Background fsync task** (for EverySec policy):
```rust
async fn fsync_loop(wal: Arc<Mutex<WriteAheadLog>>) {
    let mut interval = tokio::time::interval(Duration::from_secs(1));
    loop {
        interval.tick().await;
        let mut wal = wal.lock();
        if wal.pending_count > 0 {
            wal.file.sync_all().await.ok();
            wal.pending_count = 0;
        }
    }
}
```

### Snapshot Strategy

For periodic snapshots, serialize the full keyspace state:

1. Clone the current state (Rust's `Arc` makes this cheap for reference-counted data)
2. Spawn a background task to write the snapshot
3. On recovery, load snapshot first, then replay AOF commands after the snapshot timestamp

### Recovery Process

1. Load latest snapshot (if exists)
2. Open AOF file, seek to first command after snapshot timestamp
3. Replay each command against the in-memory state
4. Ready to accept connections

### MQ-Specific Optimization

For queue workloads, you can optimize persistence:
- Only log state-changing commands (skip BRPOP that returns nil, skip reads)
- Batch multiple ZADD/LPUSH operations into a single WAL entry
- Compact the AOF by snapshotting just the current queue state (much smaller than replaying all historical job additions)

## 7. Smart Optimizations

### Command Pipeline Detection

When a client sends multiple commands in one TCP read, batch them:

```rust
async fn process_pipeline(
    frames: Vec<Frame>,
    state: &ServerState,
    ctx: &mut ConnectionContext,
) -> Vec<Frame> {
    let mut responses = Vec::with_capacity(frames.len());
    for frame in frames {
        let cmd = Command::from_frame(frame)?;
        responses.push(state.execute(ctx, cmd).await);
    }
    responses
}
```

DragonflyDB's `MultiCommandSquasher` takes this further - if all pipeline commands target the same shard, execute them as a batch without per-command dispatch overhead.

### BullMQ Pattern Recognition

BullMQ has predictable command sequences. The `moveToActive` pattern always:
1. `EVALSHA <sha1> 11 <keys...> <args...>`

If you detect a known BullMQ script SHA1, execute the optimized Rust path directly:

```rust
fn is_known_bullmq_script(sha1: &str) -> Option<BullMqOperation> {
    match sha1 {
        "abc123..." => Some(BullMqOperation::MoveToActive),
        "def456..." => Some(BullMqOperation::MoveToFinished),
        "ghi789..." => Some(BullMqOperation::AddStandardJob),
        _ => None,
    }
}
```

**Caution**: BullMQ updates scripts between versions. Hash the bundled scripts at startup and detect dynamically, do not hardcode SHA1 values.

### Script Pre-Loading

At server startup, load and pre-compile all known BullMQ scripts:

```rust
async fn preload_bullmq_scripts(engine: &LuaEngine) {
    let scripts_dir = include_dir!("bundled/bullmq/commands");
    for file in scripts_dir.files() {
        if file.path().extension() == Some("lua") {
            let content = file.contents_utf8().unwrap();
            let sha1 = hex::encode(Sha1::digest(content.as_bytes()));
            engine.cache_script(&sha1, content);
        }
    }
}
```

### Connection-Local Caching

If a worker always reads from the same queue, cache the shard reference:

```rust
struct ConnectionContext {
    peer: SocketAddr,
    auth: bool,
    db: u8,
    name: Option<String>,
    /// Cached shard for frequently-accessed keys
    hot_shard: Option<(Bytes, usize)>,
}
```

### Blocking Operation Optimization

BRPOP is the hot path for Sidekiq and Celery workers. Optimize the wake-up chain:

```rust
struct BlockingManager {
    /// Key -> list of waiting clients (FIFO order)
    waiters: DashMap<Bytes, VecDeque<oneshot::Sender<Bytes>>>,
}

impl BlockingManager {
    /// Called when LPUSH/RPUSH adds to a key
    fn notify_push(&self, key: &Bytes, value: Bytes) {
        if let Some(mut waiters) = self.waiters.get_mut(key) {
            if let Some(sender) = waiters.pop_front() {
                let _ = sender.send(value);
            }
        }
    }

    /// Called by BRPOP handler
    async fn wait_for_key(
        &self,
        key: Bytes,
        timeout: Duration,
    ) -> Option<Bytes> {
        let (tx, rx) = oneshot::channel();
        self.waiters
            .entry(key)
            .or_default()
            .push_back(tx);
        tokio::time::timeout(timeout, rx).await.ok()?.ok()
    }
}
```

### Memory Pooling

Pre-allocate response buffers to reduce allocation pressure:

```rust
struct ResponsePool {
    pool: crossbeam_queue::ArrayQueue<BytesMut>,
}

impl ResponsePool {
    fn get(&self) -> BytesMut {
        self.pool.pop().unwrap_or_else(|| BytesMut::with_capacity(4096))
    }

    fn put(&self, mut buf: BytesMut) {
        buf.clear();
        let _ = self.pool.push(buf);
    }
}
```

## 8. Existing Projects and Lessons

### mini-redis (Tokio Reference)

**What to learn**: Clean RESP parsing, connection handling, shared state with std::sync::Mutex, Semaphore-based connection limits, graceful shutdown via broadcast channel + MPSC drain.

**What it lacks**: No EVAL, no sorted sets, no streams, no persistence, no MULTI/EXEC.

### DragonflyDB (Performance Leader)

**What to learn**: Shared-nothing with ProactorPool, fiber-based concurrency, `MultiCommandSquasher` for pipeline batching, EngineShardSet for key distribution, thread-local ServerState.

**Key insight**: Their 25x throughput gain comes primarily from multi-threading, not algorithmic improvements. Their memory efficiency comes from not forking for snapshots.

### Garnet (Microsoft)

**What to learn**: Tsavorite storage engine separates parsing from storage. Two-phase locking for multi-key transactions. Lua scripting alongside C# extensibility. "Narrow-waist" API design pattern.

**Key insight**: Sub-300us p99.9 latency through network-thread-local processing (TLS and storage on same thread).

### Kvrocks (Apache)

**What to learn**: How to implement Redis compatibility with a different storage backend (RocksDB). Namespace isolation pattern. Centralized cluster management (simpler than gossip).

**Key insight**: RocksDB persistence is much simpler than implementing your own. Consider it as a persistence backend if you need disk-backed durability.

### redcon/redcon.rs (RESP Frameworks)

**What to learn**: Minimal RESP server framework pattern. Callback-based command dispatch. The Go version achieves 2x+ Redis throughput, proving that a simple RESP server can outperform Redis.

## 9. Client Compatibility Testing

### Replay-Based Testing

Record production Redis traffic and replay against your server:

```bash
# Record with redis-cli monitor (or tcpdump)
redis-cli MONITOR > redis_traffic.log

# Or use a proxy that records RESP commands
# Then replay against your server
```

### BullMQ Integration Test

```typescript
// Test with actual BullMQ client
const queue = new Queue('test', {
  connection: { host: 'localhost', port: 6380 } // your server
});

// Test core operations
await queue.add('job1', { data: 'test' });
await queue.add('job2', { data: 'test' }, { delay: 5000 });
await queue.add('job3', { data: 'test' }, { priority: 1 });

const worker = new Worker('test', async (job) => {
  return { processed: true };
}, { connection: { host: 'localhost', port: 6380 } });

// Verify: jobs complete, events fire, delayed jobs promote
```

### Sidekiq Integration Test

```ruby
Sidekiq.configure_client do |config|
  config.redis = { url: 'redis://localhost:6380' }
end

class TestWorker
  include Sidekiq::Job
  def perform(data)
    # Process
  end
end

TestWorker.perform_async('test')
TestWorker.perform_in(5.seconds, 'scheduled')
```

### Version Negotiation

Some clients send `HELLO 3` (RESP3), others use `HELLO 2`, and older clients skip HELLO entirely:

```rust
fn handle_hello(args: &[Bytes]) -> Frame {
    let proto = args.get(0)
        .and_then(|b| std::str::from_utf8(b).ok())
        .and_then(|s| s.parse::<u32>().ok())
        .unwrap_or(2);

    if proto == 3 {
        // Return RESP3 map response
        // Switch connection to RESP3 mode
    } else if proto == 2 {
        // Return RESP2 array response
    } else {
        Frame::Error("NOPROTO unsupported protocol version".into())
    }
}
```

### CLIENT SETNAME / CLIENT ID

ioredis (used by BullMQ) sends `CLIENT SETNAME` early in the connection. Track this for debugging:

```rust
struct ConnectionContext {
    id: u64,          // Monotonic connection ID
    name: Option<String>,
    protocol: RespVersion,
    authenticated: bool,
    db: u8,
}
```

## 10. Performance Targets and Benchmarking

### Target Metrics

| Operation | Latency Target | Throughput Target |
|-----------|---------------|------------------|
| SET/GET | p99 < 0.5ms | 200K+ ops/sec |
| LPUSH/RPUSH | p99 < 0.5ms | 200K+ ops/sec |
| BRPOP (with data) | p99 < 1ms | 100K+ ops/sec |
| ZADD/ZRANGEBYSCORE | p99 < 1ms | 150K+ ops/sec |
| EVAL (simple script) | p99 < 2ms | 50K+ ops/sec |
| EVAL (BullMQ moveToActive) | p99 < 5ms | 20K+ ops/sec |
| Pipeline (16 cmds) | p99 < 2ms | 1M+ ops/sec |

### Benchmarking Tools

```bash
# Basic throughput
redis-benchmark -h localhost -p 6380 -c 50 -n 1000000 -t set,get,lpush,lpop -q

# With pipelining
redis-benchmark -h localhost -p 6380 -c 50 -n 1000000 -P 16 -t set,get -q

# Custom command (EVAL)
redis-benchmark -h localhost -p 6380 -n 100000 \
  eval "return redis.call('set',KEYS[1],ARGV[1])" 1 __key__ __val__

# memtier_benchmark (more realistic workloads)
memtier_benchmark -s localhost -p 6380 --ratio=1:1 \
  --pipeline=16 -c 50 -t 4 --test-time=30
```

### Memory Efficiency

For queue workloads, your server should use significantly less memory than Redis:
- No need for Redis's complex encoding layers (listpack, ziplist, intset)
- Queue jobs are transient - aggressive cleanup of completed/failed sets
- Arena allocation for batch operations
- No fork for persistence (no 2x memory spike during BGSAVE)

Target: **50-70% of Redis memory usage** for the same queue workload.

## Common Pitfalls

| Pitfall | Why It Happens | How to Avoid |
|---------|---------------|--------------|
| RESP parsing off-by-one | Missing CRLF, wrong length count | Fuzz test with `cargo-fuzz`, test with real clients |
| Lua nil vs false | Redis converts nil to false in Lua | Always return `false` for nil, never Lua `nil` |
| BRPOP starvation | One queue always has data, others starve | Rotate queue check order (like Sidekiq shuffle) |
| Sorted set precision | f64 score comparison issues | Use integer millisecond timestamps, not floats |
| Script atomicity | Other commands interleave during Lua exec | Hold shard lock for entire script duration |
| Memory leak on streams | XADD without XTRIM | Implement MAXLEN on streams, default to 10K |
| Pipeline response ordering | Responses sent out of order | Process pipeline commands sequentially per connection |
| EVALSHA cold start | Client sends EVALSHA before any EVAL | Pre-load known scripts or return NOSCRIPT correctly |
| CONNECTION_RESET on shutdown | Abrupt TCP close | Send -ERR on shutdown, drain connections |
| BullMQ maxmemory-policy | BullMQ checks CONFIG GET maxmemory-policy | Return `noeviction` from CONFIG GET |

## Best Practices

Synthesized from 42 sources:

1. **Start with RESP2, add RESP3 later** - most MQ clients still use RESP2. RESP3 is only needed for modern clients that send HELLO 3.

2. **Use sharded state from day one** - even if starting single-threaded, shard the keyspace. It is much harder to add sharding later.

3. **Implement EVAL before EVALSHA** - BullMQ falls back to EVAL on NOSCRIPT. Get full script execution working first, then add caching.

4. **Test with real clients, not just redis-benchmark** - BullMQ's Lua scripts are the real compatibility test. Run the BullMQ test suite against your server.

5. **Log unsupported commands instead of crashing** - return `-ERR unknown command` and log what was sent. This helps identify missing command coverage.

6. **Use bytes::Bytes everywhere** - zero-copy reference counting eliminates most allocation overhead in the hot path.

7. **Separate hot path from cold path** - BRPOP/LPUSH/EVAL are hot. INFO/CONFIG/CLIENT are cold. Optimize accordingly.

8. **Profile Lua execution separately** - Lua scripts can dominate latency. Benchmark your mlua integration in isolation before integrating.

9. **Implement INFO early** - monitoring tools and clients check INFO for health. Return at minimum: server version, connected_clients, used_memory.

10. **Design for zero external dependencies** - the whole point is replacing Redis. Ship a single binary that starts with `./mq-server --port 6380`.

## Recommended Crate Dependencies

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
bytes = "1"
mlua = { version = "0.10", features = ["lua51", "send"] }
sha1 = "0.10"
dashmap = "5"
parking_lot = "0.12"
ahash = "0.8"
crossbeam-skiplist = "0.1"
slab = "0.4"
crc = "3"
tokio-rustls = "0.26"
tracing = "0.1"
tracing-subscriber = "0.3"
clap = { version = "4", features = ["derive"] }

[dev-dependencies]
redis = { version = "0.25", features = ["tokio-comp"] }
```

## Further Reading

| Resource | Type | Why Recommended |
|----------|------|-----------------|
| [Redis Protocol Spec](https://redis.io/docs/latest/develop/reference/protocol-spec/) | Official Docs | Definitive RESP reference |
| [mini-redis](https://github.com/tokio-rs/mini-redis) | Code | Best Rust RESP server reference |
| [BullMQ Lua Scripts](https://github.com/taskforcesh/bullmq/tree/master/src/commands) | Code | The 51 scripts you must support |
| [mlua docs](https://docs.rs/mlua/latest/mlua/) | Docs | Lua embedding API reference |
| [DragonflyDB Architecture](https://github.com/dragonflydb/dragonfly) | Code | Multi-threaded Redis architecture |
| [Garnet](https://github.com/microsoft/garnet) | Code | Tsavorite engine, Lua support patterns |
| [Redis Lua Scripting](https://redis.io/docs/latest/develop/interact/programmability/eval-intro/) | Official Docs | EVAL/EVALSHA semantics |
| [Redis Persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/) | Official Docs | AOF/RDB design reference |
| [Tokio Tutorial](https://tokio.rs/tokio/tutorial) | Tutorial | Async Rust patterns for servers |
| [redis-protocol crate](https://docs.rs/redis-protocol/) | Docs | RESP parsing library |
| [Sidekiq Internals](https://github.com/sidekiq/sidekiq) | Code | Simple MQ Redis usage patterns |
| [Celery/Kombu Transport](https://github.com/celery/kombu) | Code | Python MQ Redis patterns |
| [Redis Benchmarks](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/benchmarks/) | Official Docs | Benchmarking methodology |
| [crossbeam-skiplist](https://docs.rs/crossbeam-skiplist/) | Docs | Concurrent sorted set impl |
| [Kvrocks](https://github.com/apache/kvrocks) | Code | Redis compat on different storage |

---

*Generated by /learn from 42 sources.*
*See `resources/resp-mq-server-rust-sources.json` for full source metadata.*

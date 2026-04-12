# feapp API

Complete reference for the `feapp.*` API surface available to app code.

---

## Actors

Each API namespace is available to specific actors. Calling a namespace not available to your actor throws `feapp.PermissionError`.

| Namespace | Frontend | Stateful Worker | Stateless Worker |
| --- | --- | --- | --- |
| [feapp.storage](#feappstorage) | read + write | read + write | read only (get, list) |
| [feapp.ipc](#feappipc) | send, on, reconnect | broadcast, sendTo, on | — |
| [feapp.sessions](#feappsessions) | — | full access | — |
| [feapp.worker](#feappworker) | stateful status + diagnose | — | — |
| [feapp.stateless](#feappstateless) | invoke | — | exports |
| [feapp.schedule](#feappschedule) | — | full access | — |
| [feapp.fs](#feappfs) | — | full access | — |
| [feapp.log](#feapplog) | — | full access | full access |
| [feapp.app](#feappapp) | full access | — | — |
| [feapp.permissions](#feapppermissions) | full access | — | — |

---

## feapp.storage

The only durable storage in a `.feapp` app. Backed by the remoteStorage protocol. Syncs across devices. Source of truth for all persistent data.

Browser storage (IndexedDB, localStorage, sessionStorage, cookies, Cache API, OPFS) is ephemeral — managed and namespaced by the runner, cleared on every profile switch. Never use browser storage for persistent data.

### Core operations

```javascript
// Read a value. Returns null if path does not exist.
const value = await feapp.storage.get(path)

// Write a value.
await feapp.storage.set(path, value)
await feapp.storage.set(path, value, { throwOnConflict: true })
await feapp.storage.set(path, value, { throwIfExists: true })

// Delete a value.
await feapp.storage.delete(path)
await feapp.storage.delete(path, { throwOnConflict: true })

// List children of a path.
const children = await feapp.storage.list(path)
const all = await feapp.storage.list(path, { recursive: true })

// Watch a path for changes.
const unwatch = feapp.storage.watch(path, handler)
unwatch()
```

### Watch handler

```javascript
feapp.storage.watch('/myapp/articles/', (event) => {
  event.path      // exact path that changed
  event.newValue  // new value, null if deleted
  event.origin    // 'local' | 'remote'
})
```

`origin: 'local'` — change was made by this running instance.  
`origin: 'remote'` — change came from another device or instance.

### list() return values

```javascript
await feapp.storage.list('/myapp/')
// → ['articles/', 'settings']
// trailing slash = directory. no trailing slash = file.

await feapp.storage.list('/myapp/', { recursive: true })
// → ['articles/123', 'articles/456', 'settings']
// recursive returns file paths only, no directories
```

### Conditional writes

Every stored value has an ETag — a version token assigned by the remoteStorage server.

**throwOnConflict** — rejects the write if another writer committed since the last read. Use for data where silent overwrite would cause loss.

```javascript
try {
  await feapp.storage.set('/myapp/db', newDb, { throwOnConflict: true })
} catch (e) {
  if (e instanceof feapp.storage.ConflictError) {
    const current = await feapp.storage.get('/myapp/db')
    const merged = merge(current, newDb)
    await feapp.storage.set('/myapp/db', merged, { throwOnConflict: true })
  }
}
```

**throwIfExists** — rejects the write if any value already exists at the path. Atomic initialization — no race between checking and writing.

```javascript
try {
  await feapp.storage.set('/myapp/config', defaults, { throwIfExists: true })
} catch (e) {
  if (e instanceof feapp.storage.ExistsError) { /* already initialized */ }
}
```

### ETag as lock primitive

There is no lock API. The ETag is the lock primitive. Use the retry pattern for mutual exclusion:

```javascript
async function atomicUpdate(path, updateFn) {
  while (true) {
    const current = await feapp.storage.get(path)
    const updated = updateFn(current)
    try {
      await feapp.storage.set(path, updated, { throwOnConflict: true })
      return
    } catch (e) {
      if (e instanceof feapp.storage.ConflictError) continue
      throw e
    }
  }
}
```

### Path conventions

```
Paths begin with /
Paths ending with /   are directories (have children, no direct value)
Paths not ending with / are files (have a value)

/feapp/   is reserved for runner internal use — do not use
          see MANIFEST.md reserved_path_override for override options
```

### Data design guidance

```javascript
// Good — one file per entity, no conflict between entities
await feapp.storage.set('/articles/123', article)
await feapp.storage.set('/articles/456', article)

// Bad — all entities in one file, every write conflicts
await feapp.storage.set('/articles', { 123: article, 456: article })

// Good — list is the index
const ids = await feapp.storage.list('/articles/')

// Bad — separate index file that must be kept in sync
await feapp.storage.set('/article-index', [123, 456])

// Good — SQLite blob for transactional consistency
const raw = await feapp.storage.get('/myapp/db')
const db = raw ? new SQL.Database(raw) : new SQL.Database()
db.run('BEGIN')
// ... transactional operations
db.run('COMMIT')
await feapp.storage.set('/myapp/db', db.export(), { throwOnConflict: true })
```

### Access by actor

```
Frontend:          read + write
Stateful worker:   read + write
Stateless worker:  get + list only
                   watch(), set(), delete() throw feapp.storage.PermissionError
```

### Error reference

```javascript
feapp.storage.ConflictError     // ETag mismatch — another writer committed first
                                // only thrown when throwOnConflict: true
                                // default is last-write-wins

feapp.storage.ExistsError       // path already has a value
                                // only thrown when throwIfExists: true

feapp.storage.OfflineError      // remoteStorage server unreachable
                                // fires regardless of topology when server is down

feapp.storage.PermissionError   // path outside declared manifest permissions
                                // or operation not permitted for this actor

feapp.storage.RateLimitError    // server returned 429
  e.retryAfter                  // seconds until retry is safe

feapp.storage.StorageFullError  // server returned 507
```

---

## feapp.ipc

Persistent bidirectional channel between the frontend and the stateful worker. The transport mechanism is implementation-defined. See WIRE.md B3 for the wire contract.

The frontend is always the hub. The stateful worker is always the spoke. There is no worker-to-worker communication.

### Frontend side

```javascript
// Send an event to the stateful worker.
// Guaranteed delivery or throws.
// No ordering guarantee across concurrent sends.
// Await sequentially to guarantee order.
await feapp.ipc.send(event, payload)

// Register a handler for events from the stateful worker.
feapp.ipc.on(event, (event) => {
  event.data    // the payload
})

// Manually trigger a reconnect attempt.
// Call after catching an IPCError or after observing
// feapp.worker.stateful.status become 'unreachable'.
feapp.ipc.reconnect()
```

`feapp.ipc.send()` throws if the stateful worker status is not `'running'`. Catch the error and check `feapp.worker.stateful.status` to determine the cause.

> **STUB** — The error type thrown on delivery failure is not yet named. See TODOS.md item 13.

### Stateful worker side

```javascript
// Send to all currently connected sessions.
feapp.ipc.broadcast(event, payload)

// Send to one specific session.
feapp.ipc.sendTo(sessionId, event, payload)

// Receive events from any connected frontend session.
feapp.ipc.on(event, (event) => {
  event.data       // the payload
  event.sessionId  // which session sent this
  event.reply(responseEvent, payload)  // reply to this sender only
                                        // sugar for sendTo(event.sessionId, ...)
})
```

### Delivery model

```
send()          guaranteed delivery or throws
                the runner acknowledges receipt before the promise resolves
                messages in flight when channel drops are not retried — they throw

ordering        no ordering guarantee across concurrent sends
                await sequentially to guarantee order:

                await feapp.ipc.send('a', {})  // arrives first
                await feapp.ipc.send('b', {})  // arrives second

reconnect       on channel drop, status becomes 'unreachable'
                developer calls feapp.ipc.reconnect() to attempt reconnection
                reconnect creates a new session — previous session is gone
                messages sent before reconnect that threw are not replayed
```

---

## feapp.sessions

Available to the stateful worker only. Tracks all currently connected frontend sessions.

```javascript
// Get all currently connected sessions.
feapp.sessions.list()   // returns Session[]

// React to sessions connecting and disconnecting.
feapp.sessions.on('connect', (session) => { ... })
feapp.sessions.on('disconnect', (session) => { ... })
```

### Session object

```javascript
session.id           // string — opaque stable identifier for this connection
session.connectedAt  // Date — when this session connected
```

Session IDs are used by `feapp.ipc.sendTo()` and available in IPC event handlers via `event.sessionId`. They are stable for the lifetime of a connection. A reconnect creates a new session with a new ID.

---

## feapp.worker

Available to the frontend only. Exposes the stateful worker's connection status.

```javascript
// Current status — passive, always available.
feapp.worker.stateful.status
// 'starting'     — runner is launching the worker, not yet reachable
// 'running'      — IPC channel is up, send() is safe
// 'unreachable'  — channel is down for any reason (network, crash, stopped)

// React to status changes.
feapp.worker.stateful.on('status-change', (status) => { ... })

// Active probe — call after observing 'unreachable' to diagnose cause.
const result = await feapp.worker.stateful.diagnose()
// result.connectivity   boolean — can the runner reach the worker endpoint?
// result.worker         'running'            — worker is up and responding
//                       'crashed'            — worker process exited unexpectedly
//                       'storage-unavailable'— worker running, cannot reach remoteStorage
//                       'starting'           — worker still initializing
//                       'unknown'            — reachable but not responding
```

`status` describes the channel as the frontend observes it. `diagnose()` is an active probe that gives a more precise picture of why the channel is down. Call `diagnose()` deliberately — it makes network requests.

---

## feapp.stateless

On-demand computation. The frontend invokes an exported function in the stateless worker by name and awaits the result. The stateless worker is always available — there is no lifecycle to manage.

### Frontend side

```javascript
const result = await feapp.stateless.invoke(name, args)
// args must be JSON-serializable
// result must be JSON-serializable
```

### Stateless worker side

Any exported async function is a callable endpoint. Non-exported functions are not reachable.

```javascript
export async function summarize({ text, maxWords = 100 }) {
  // ...
  return { summary }
}

export async function estimateReadTime({ text }) {
  const words = text.trim().split(/\s+/).length
  return { minutes: Math.ceil(words / 200) }
}
```

### External invocation (webhooks)

The stateless worker may be invoked by external HTTP requests in addition to frontend calls. This enables webhook integrations — external services posting directly to the app's stateless worker. When invoked externally, `session_id` in the wire envelope is null. The function signature is identical; the function does not know how it was invoked.

> **Note** — Authentication and security implications of external stateless invocation are not yet fully specified. See TODOS.md item 7.

### Error model

```javascript
try {
  const result = await feapp.stateless.invoke('fn', args)
} catch (e) {
  if (e instanceof feapp.stateless.NotFoundError) {
    // function name does not exist in stateless worker module
  }
  if (e instanceof feapp.stateless.ExecutionError) {
    // function threw internally
    console.error(e.cause)  // the original error
  }
}
```

Both `NotFoundError` and `ExecutionError` extend `ExecutionError` for catch-all handling.

---

## feapp.schedule

Available to the stateful worker only. Registers recurring callbacks that execute regardless of whether any frontend is connected.

### feapp.schedule.every()

```javascript
feapp.schedule.every(interval, fn, options)
```

`every()` requires a persistent process between invocations — something must remain running to measure the gap and trigger the next invocation. Runners that cannot provide a persistent stateful worker process cannot implement `every()`. Such runners must refuse to open any `.feapp` file that declares a stateful worker, the same as refusing an unsupported spec version.

`interval` — string duration: `'1s'`, `'30s'`, `'5m'`, `'1h'`, `'1d'`.

> **STUB** — Minimum interval value (e.g. 1s) not yet confirmed. See TODOS.md item 7.

`options.timeout` — string duration. Strongly recommended. Runner kills the invocation on timeout, logs via `feapp.log.error`, and starts the next interval gap.

> **STUB** — Whether timeout is required (CLI error) or recommended (CLI warning) for every() is not yet confirmed. See TODOS.md item 6.

`options.max_concurrent` — integer, default `1`.

**Behavior with max_concurrent: 1 (default):**
The interval is the gap between the end of one invocation and the start of the next. Pile-up is impossible.

```
fn starts → fn runs 5 min → fn ends → [30m gap] → fn starts again
```

**Behavior with max_concurrent > 1:**
The interval becomes clock-based. If all slots are full when the interval fires, that firing is skipped.

### feapp.schedule.cron()

```javascript
feapp.schedule.cron(expression, fn, options)
```

`expression` — standard 5-field cron syntax: `'0 3 * * *'`.

`options.timeout` — required. CLI error if absent. Runner kills hung invocations.

`options.max_concurrent` — integer, default `1`. If all slots full when the cron time arrives, that firing is skipped.

Cron fires in the timezone configured by the runner. Timezone is runner-defined — not a per-schedule option.

### feapp.schedule.on()

```javascript
feapp.schedule.on(event, fn)
// event: 'worker-start' | 'storage-connected'
```

`'worker-start'` — fires once when the worker module first loads.  
`'storage-connected'` — fires once after storage is confirmed reachable.

Note: `'app-open'` and `'app-close'` events do not exist. The stateful worker lifecycle is independent of the frontend. Use `feapp.sessions.on('connect', ...)` to detect frontend connections.

---

## feapp.fs

Available to the stateful worker only. Provides access to filesystem paths declared in the manifest under `workers.stateful.permissions.filesystem`.

Paths are abstract — declared in the manifest and mapped to real filesystem locations by the runner at launch. The worker never sees real paths and never constructs paths containing system locations or user identity.

```javascript
feapp.fs.read(path)              // returns Uint8Array
feapp.fs.write(path, data)       // data: Uint8Array | string
feapp.fs.list(path)              // returns string[] of immediate children
feapp.fs.delete(path)
feapp.fs.watch(path, handler)    // returns unwatch function
feapp.fs.exists(path)            // returns boolean
```

Access outside declared manifest paths throws `feapp.fs.PermissionError`.

> **STUB** — The following aspects of feapp.fs are not yet specified and need depth:
> - Large file handling and size limits
> - Streaming API (vs Uint8Array for large files)
> - Memory constraints and what happens when a read/write exceeds available memory
> - Buffer ownership — is the Uint8Array from read() a copy or a view?
> - Concurrent access — behavior when two schedule callbacks write the same path simultaneously
>
> See TODOS.md item 3.

---

## feapp.log

Available to both worker types. The runner captures all log output and surfaces it in library UI. Logs are never exposed inside a `.feapp` frontend.

```javascript
feapp.log.info(message, context?)
feapp.log.warn(message, context?)
feapp.log.error(message, error?)

// Examples
feapp.log.info('feed sync complete', { feedCount: 42, newItems: 7 })
feapp.log.warn('rate limited', { retryAfter: 60 })
feapp.log.error('storage write failed', err)
```

Log output is essential for debugging apps long after the original developer is unreachable. Log generously.

---

## feapp.app

Available to the frontend only. Read-only metadata about the running app and current session.

```javascript
feapp.app.id        // string — 'com.developer.feedreader'
feapp.app.name      // string — 'Feed Reader'
feapp.app.version   // string — '1.0.0'
feapp.app.profile   // string — opaque stable identifier for this account+app combination
                    // not user identity — contains no email, username, or endpoint
                    // stable for the lifetime of a profile
                    // changes when the user switches accounts
                    // may be used to namespace browser storage keys as additional safety
```

---

## feapp.permissions

> **STUB** — Not yet designed. See TODOS.md item 2.
>
> `feapp.permissions` will provide a runtime API for requesting additional permissions
> declared in the manifest but not yet granted by the user. The design needs to specify:
> - Call shape
> - Whether the app pauses during the user confirmation dialog
> - Which component restarts on approval
> - Whether workers can call it or only the frontend
> - How the grant flows through the ecosystem

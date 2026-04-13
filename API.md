# feapp API

Complete reference for the `feapp.*` API surface available to app code.

---

## Actors

Each API namespace is available to specific actors. Calling a namespace not available to your actor throws `feapp.PermissionError`.

| Namespace | Frontend | Stateful Worker | Stateless Worker |
| --- | --- | --- | --- |
| [feapp.storage](#feappstorage) | read + write | read + write | read + write |
| [feapp.ipc](#feappipc) | send, on, reconnect | broadcast, sendTo, on | — |
| [feapp.sessions](#feappsessions) | — | full access | — |
| [feapp.worker](#feappworker) | stateful status + diagnose | — | — |
| [feapp.stateless](#feappstateless) | invoke | — | exports |
| [feapp.schedule](#feappschedule) | — | full access | — |
| [feapp.permissions](#feapppermissions) | read-only | read-only | read-only |
| [feapp.log](#feapplog) | — | full access | full access |
| [feapp.app](#feappapp) | full access | full access | full access |

---

## Execution Model

### Stateful worker

The stateful worker is a **long-lived JavaScript process with an event loop**.

**Module-level code** executes once when the worker starts. Module-level variables persist for the lifetime of the worker process. This is the actor's private memory — not durable, but persistent across events within a single stretch of uptime.

**The standard JavaScript event loop** is available. `setTimeout`, `setInterval`, `queueMicrotask`, `Promise` — all work as expected. The worker stays alive as long as the runner keeps it alive.

**`feapp.schedule.on('worker-start')`** fires after module-level code completes and storage is confirmed connected. Use this for initialization that requires storage, network, or IPC. Module-level code should be limited to variable declarations and non-async setup. This event fires on every start, including restarts after healthcheck kills — this is where WebSocket connections to external services should be established so they are re-opened after any restart.

**`setInterval` vs `feapp.schedule.every()`**: both work, different purposes. `setInterval` is fine for internal timing within a single stretch of uptime — retry backoff, debouncing, periodic in-memory cleanup. `feapp.schedule.every()` is the right tool for recurring work that matters — the runner tracks it, enforces timeouts, and surfaces it in library UI. `setInterval` is opaque to the runner and lost on restart. `feapp.schedule.every()` is re-registered when the worker restarts.

**WebSocket client** is guaranteed in the stateful worker environment. The worker can open outgoing WebSocket connections to any address in its granted network set. These connections live as long as the worker does. Reconnection is the developer's responsibility — the `worker-start` event is the right place to establish connections.

**WebAssembly** is guaranteed in the stateful worker environment. WASM modules are subject to the same healthcheck as JavaScript — a WASM computation that blocks the event loop will trigger a kill.

```javascript
// Example: stateful worker with a persistent WebSocket connection
let fsService = null

feapp.schedule.on('worker-start', async () => {
  const config = feapp.permissions.configured('filesystem')
  if (config && config.endpoints.length > 0) {
    fsService = new WebSocket(config.endpoints[0])
    fsService.onmessage = async (msg) => {
      const event = JSON.parse(msg.data)
      await feapp.storage.set(`/files/${event.id}`, event.metadata)
      feapp.ipc.broadcast('file-changed', event)
    }
    fsService.onclose = () => {
      feapp.log.warn('filesystem service disconnected')
      fsService = null
    }
  }
})

feapp.schedule.every('5m', async () => {
  const feeds = await feapp.storage.list('/feeds/')
  for (const feed of feeds) {
    // ... fetch and process
  }
  feapp.ipc.broadcast('feeds-updated', { count: feeds.length })
}, { timeout: '2m' })
```

### Stateless worker

The stateless worker is a set of **on-demand functions**. Each invocation is a cold start — a fresh module context with no state carried over from any previous call.

Module-level code (imports, constant declarations) executes on each invocation. Module-level variables do not persist between calls. The runner must guarantee this — even if it keeps the module warm for performance, it must clear module-level state before each invocation.

There is no WebSocket support — invocations are short-lived and there is nothing to hold a connection open. `fetch()` is available for one-shot request/response operations with external services.

**WebAssembly** is guaranteed in the stateless worker environment. Same healthcheck rules apply.

STUB: stateless worker example needs revision for cases where summary is longer than the default 1m timeout
```javascript
// Example: stateless worker
export async function summarize({ text, maxWords = 100 }) {
  const data = await feapp.storage.get('/config/summarizer')
  // ... compute summary using config
  return { summary, wordCount }
}

export async function processWithAI({ prompt }) {
  const config = feapp.permissions.configured('llm')
  if (!config || config.endpoints.length === 0) {
    throw new Error('No AI endpoint configured')
  }
  const response = await fetch(config.endpoints[0] + '/api/generate', {
    method: 'POST',
    body: JSON.stringify({ prompt })
  })
  const result = await response.json()
  await feapp.storage.set(`/ai/results/${Date.now()}`, result)
  return result
}
```

---

## Healthcheck

The runner monitors both worker types with a periodic healthcheck that detects a blocked event loop.

**Mechanism:** The runner pings the worker on an internal control channel every 1 second. The runtime responds automatically — this is not a developer-facing API. The developer's code never sees the ping.

**Threshold:** If the response does not arrive within 10 seconds, the event loop is blocked. The default threshold of 10 seconds is configurable in the library.

**On failure — stateful worker:** The runner kills the worker process and initiates a restart with backoff. All in-memory state is lost. `feapp.storage` is the only state that survives.

**On failure — stateless worker:** The runner kills the specific invocation. `feapp.stateless.invoke()` rejects with a `feapp.stateless.ExecutionError` on the frontend side.

**What this means for developers:** Your code must yield to the event loop. No synchronous loop that runs for more than a few seconds. No tight WASM computation that never returns. If you have heavy computation, delegate it to an external service via `fetch()` or WebSocket. The healthcheck enforces this structurally — it is not a suggestion.

---

## Restart Policy

When the stateful worker is killed by the healthcheck or crashes, the runner restarts it with exponential backoff:

```
Attempt 1:   wait 1s, then restart
Attempt 2:   wait 10s
Attempt 3:   wait 30s
Attempt 4:   wait 1m
Attempt 5:   wait 5m
Attempt 6:   wait 10m
Attempt 7:   wait 20m
Attempt 8:   wait 40m
Attempt 9:   wait 60m
After attempt 9: disable the worker
```

When the worker is disabled, `feapp.worker.stateful.status` becomes `'disabled'`. The frontend can still open, read `feapp.storage`, and invoke stateless functions. It cannot communicate with the stateful worker. The frontend should detect the `'disabled'` status and inform the user.

The library surfaces a notification when a worker is disabled. The user can re-enable the worker from library settings, which resets the retry counter and immediately attempts a restart. If it fails again, the backoff sequence starts over from the beginning.

A successful start (worker stays alive long enough to respond to healthchecks) resets the retry counter.

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
Stateless worker:  read + write
```

All actors have full read and write access to `feapp.storage`. Concurrent writes from multiple actors are safe — `feapp.storage` provides ETags and conditional writes for conflict resolution. Use `throwOnConflict` when silent overwrite would cause data loss.

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
// 'restarting'   — worker was killed or crashed and the runner is restarting
//                  with backoff — the runner is waiting before the next attempt
// 'disabled'     — worker exhausted all restart attempts and has been disabled
//                  re-enable from library settings

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

### Frontend behavior during stateful worker downtime

When the stateful worker is in any status other than `'running'`, the frontend can still:
- Read and write `feapp.storage` directly
- Invoke stateless functions via `feapp.stateless.invoke()`
- Read permissions via `feapp.permissions`

The frontend cannot send IPC messages. `feapp.ipc.send()` will throw. The frontend should detect the status and adapt its UI accordingly.

---

## feapp.stateless

On-demand computation. The frontend invokes an exported function in the stateless worker by name and awaits the result. The stateless worker is always available and fully independent of the stateful worker.

### Frontend side

```javascript
const result = await feapp.stateless.invoke(name, args)
```

`args` — optional. Must be JSON-serializable if provided. See [Serialization contract](#serialization-contract).

`result` — the return value from the function. Must be JSON-serializable. Functions that do not explicitly return a value resolve to `undefined`.

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

### Serialization contract

Arguments and return values must be **JSON-serializable**. The following types are permitted at the boundary:

```
string
number (finite only — NaN and Infinity are not permitted)
boolean
null
plain object (all values must also be JSON-serializable)
array (all elements must also be JSON-serializable)
```

The runner validates at the boundary. If the argument or return value contains a type not in this list, `invoke()` rejects with a `feapp.stateless.SerializationError` **before** the function is called (for invalid arguments) or **before** the result is returned to the caller (for invalid return values).

The runner must not silently coerce invalid types. `undefined` nested inside a value (e.g., `{ a: undefined }`) is a `SerializationError`. Functions, symbols, `Date`, `Map`, `Set`, `Uint8Array`, and all other non-JSON types are `SerializationError`.

**Exception:** A function that returns `undefined` (no explicit `return` statement) is not a serialization error. It means "completed successfully with no result." `invoke()` resolves to `undefined`.

**Binary data:** If you need to pass binary data across the stateless boundary, base64-encode it as a string. This makes the data self-describing and keeps the serialization contract simple.

### Invocation timeout

Each stateless invocation has a timeout. The default is **1 minute**. The timeout is configurable in the library.

If a function does not return within the timeout, the runner kills the invocation and `invoke()` rejects with a `feapp.stateless.TimeoutError`. This protects against hung async operations (e.g., `fetch()` to an unresponsive external service) which do not block the event loop and therefore are not caught by the healthcheck.

### Long-running work belongs in the stateful worker

If a task takes longer than the stateless timeout — syncing a large dataset, processing a backlog of records, running a multi-step AI pipeline, batch-importing paginated API results — it belongs in the stateful worker. The pattern:

1. Frontend sends a "start job" message over `feapp.ipc`
2. Stateful worker runs the job, writing progress to `feapp.storage` (e.g., `/jobs/{id}/progress`)
3. Frontend watches the progress path via `feapp.storage.watch()` and renders a progress bar
4. Stateful worker writes the final result and broadcasts completion over IPC

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
  if (e instanceof feapp.stateless.SerializationError) {
    // argument or return value is not JSON-serializable
  }
  if (e instanceof feapp.stateless.TimeoutError) {
    // function did not return within the timeout
  }
}
```

All stateless errors extend `feapp.stateless.Error` for catch-all handling.

---

## feapp.schedule

Available to the stateful worker only. Registers recurring callbacks that execute regardless of whether any frontend is connected.

Neither `every()` nor `cron()` is the right tool for CPU-intensive computation. Schedule callbacks run on the stateful worker's event loop. If a callback blocks the event loop, the healthcheck will kill the worker. Use schedule callbacks to coordinate async operations — fetching data, writing to storage, delegating heavy work to external services.

### feapp.schedule.every()

```javascript
feapp.schedule.every(interval, fn, options)
```

`every()` requires a persistent process between invocations — something must remain running to measure the gap and trigger the next invocation. Runners that cannot provide a persistent stateful worker process cannot implement `every()`. Such runners must refuse to open any `.feapp` file that declares a stateful worker, the same as refusing an unsupported spec version.

`interval` — string duration: `'1s'`, `'30s'`, `'5m'`, `'1h'`, `'1d'`.

> **STUB** — Minimum interval value (e.g. 1s) not yet confirmed. See TODOS.md item 7.

`options.timeout` — string duration. Strongly recommended. Protects the schedule from stalled async operations — a callback that awaits a hung `fetch()` or an unresponsive WebSocket will never return, silently stopping the schedule. When the timeout fires, the runner cancels that specific callback invocation (not the worker process), logs via `feapp.log.error`, and allows the next scheduled invocation to proceed normally. The CLI emits a warning if `timeout` is absent.

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

`options.timeout` — **required**. CLI error if absent. Protects the schedule from stalled async operations. When the timeout fires, the runner cancels that specific callback invocation (not the worker process), logs via `feapp.log.error`, and allows the next scheduled invocation to proceed normally.

`options.max_concurrent` — integer, default `1`. If all slots full when the cron time arrives, that firing is skipped.

Cron fires in the timezone configured by the runner. Timezone is runner-defined — not a per-schedule option.

### feapp.schedule.on()

```javascript
feapp.schedule.on(event, fn)
// event: 'worker-start' | 'storage-connected'
```

`'worker-start'` — fires once when the worker module first loads and storage is confirmed connected. Fires on every start, including restarts. This is the right place to establish WebSocket connections to external services, initialize in-memory state from storage, and perform any setup that requires async operations.

`'storage-connected'` — fires once after storage is confirmed reachable. In practice, `worker-start` fires after `storage-connected`, so most developers should use `worker-start`.

Note: `'app-open'` and `'app-close'` events do not exist. The stateful worker lifecycle is independent of the frontend. Use `feapp.sessions.on('connect', ...)` to detect frontend connections.

---

## feapp.log

Available to both worker types only. The runner captures all log output and surfaces it in library UI. Logs are never exposed inside a `.feapp` frontend.

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

### console.log in workers

The TC55 Console API is available in both worker types. `console.log`, `console.warn`, and `console.error` are captured by the runner and surfaced alongside `feapp.log` output in library UI. They are not silently discarded. `feapp.log` is preferred because it accepts structured context, but `console` works and goes to the same place.

---

## feapp.app

Available to all actors. Read-only metadata about the running app.

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

Available to all actors. Read-only. Reflects the permissions granted by the user for this session.

The granted set is frozen at launch. It does not change during the session. If the user changes permissions in the library, the change takes effect on next launch.

### Querying permissions

```javascript
// Check a specific permission.
feapp.permissions.query(permission)
// Returns: 'granted' | 'denied'

// 'granted' — the user approved this permission, the runner is enforcing it
// 'denied'  — the user rejected this permission, or it was not declared in the manifest
```

`permission` is a string in the format `type:value`:

```javascript
feapp.permissions.query('network:https://api.example.com')  // 'granted'
feapp.permissions.query('network:http://localhost:11434')    // 'denied'
```

The permission string format is `{type}:{value}` where `type` is `network` and `value` is the address as declared in the manifest.

### Listing granted permissions

```javascript
// Everything currently granted for this session.
feapp.permissions.granted()
// Returns: string[]
// [
//   'network:https://api.example.com',
//   'network:https://feeds.example.com'
// ]
```

This includes both static permissions and the resolved values of configured permissions. The app can use this to determine what UI to show.

### Reading configured endpoints

```javascript
// Read the user-configured values for a network_configured slot.
feapp.permissions.configured(name)
// Returns: { endpoints: string[] } | null

feapp.permissions.configured('llm')
// { endpoints: ['http://localhost:11434'] }

feapp.permissions.configured('feeds')
// { endpoints: ['https://blog.example.com', '**.wordpress.com'] }

feapp.permissions.configured('sync')
// { endpoints: ['https://myserver.example.com:5984'] }

feapp.permissions.configured('nonexistent')
// null — not declared in the manifest
```

Returns `null` if the name is not declared in the manifest. Returns `{ endpoints: [] }` if the slot is declared but the user has not configured any endpoints (or rejected the permission).

### Access by actor

```
Frontend:          feapp.permissions.query(), granted(), configured()
Stateful worker:   feapp.permissions.query(), granted(), configured()
Stateless worker:  feapp.permissions.query(), granted(), configured()
```

Workers use `configured()` to discover endpoint URLs for external services at startup, without waiting for the frontend to relay them.

### Design guidance

```javascript
// Frontend: adapt the UI to the user's permission choices.
if (feapp.permissions.query('network:https://api.example.com') === 'granted') {
  showSyncButton()
} else {
  showMessage('Enable API access in your library settings to sync.')
}

// Stateful worker: discover external service endpoints at startup.
feapp.schedule.on('worker-start', async () => {
  const llm = feapp.permissions.configured('llm')
  if (llm && llm.endpoints.length > 0) {
    initAIConnection(llm.endpoints[0])
  }
})

// Stateless worker: check endpoints before calling.
export async function callAI({ prompt }) {
  const llm = feapp.permissions.configured('llm')
  if (!llm || llm.endpoints.length === 0) {
    throw new Error('No AI endpoint configured')
  }
  return await fetch(llm.endpoints[0] + '/api/generate', {
    method: 'POST',
    body: JSON.stringify({ prompt })
  }).then(r => r.json())
}
```

The app never prompts the user for permissions directly. Permission management happens in the library UI. The app reads the current state and adapts.

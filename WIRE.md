# Wire Contracts

This document defines the wire contracts required for two independently written implementations to interoperate. Each contract is versioned independently from the feapp spec version.

**Exact version match required.** Two components must declare the same wire contract version to communicate. No compatibility assumptions. No best-effort mode. This applies without exception, especially before v1.0.

---

## Contract index

```
B1  Library → Runner              launch protocol
                                  how the library spawns the runner
                                  how profile context is passed

B3  Frontend ↔ Stateful Worker    IPC channel
                                  message envelope format
                                  delivery acknowledgment
                                  session routing

B4  Any actor ↔ remoteStorage     inherited protocol
                                  not defined here — referenced
```

B2 (Desktop Library ↔ Cloud Library) is defined in the [ecosystem spec](https://github.com/openfeapp/ecosystem-spec/blob/main/WIRE.md). It is an ecosystem-level contract, not a `.feapp` format concern.

---

## B1 — Library → Runner Launch Protocol

The library spawns the runner as a subprocess and passes profile context to it. The runner never calls the library.

### Wire contract version

> **STUB** — Wire contract versioning mechanism not yet specified. See TODOS.md item 11.

### Launch invocation

```
feapp-runner [options] <path-to-feapp-file>
```

> **STUB** — The exact CLI argument format and option set are not yet confirmed. The transport for profile context (stdin JSON proposed) needs confirmation. See TODOS.md item 4.

### Profile context

The profile context passed from library to runner contains:

```json
{
  "feapp_wire": "0.1",
  "app_id": "com.developer.feedreader",
  "app_version": "1.0.0",
  "storage_endpoint": "https://storage.example.com",
  "storage_token": "bearer-token",
  "worker_endpoint": "wss://worker.example.com/stateful",
  "stateless_endpoint": "https://worker.example.com/stateless",
  "storage_namespace": "/feedreader/",
  "profile_id": "opaque-profile-identifier"
}
```

The runner uses these values to:
- Construct all storage path namespacing
- Inject worker endpoint URLs and auth into the frontend runtime
- Configure worker process credentials

The worker never receives raw profile context. The runner applies it transparently.

### Version mismatch behavior

If the runner does not support the declared `feapp_wire` version, it must exit with a non-zero code and write a human-readable error to stderr explaining which wire version is required.

---

## B3 — Frontend ↔ Stateful Worker IPC Channel

Defines the message envelope format for the IPC channel between the frontend and the stateful worker. The transport mechanism is implementation-defined.

### Wire contract version

> **STUB** — Wire contract versioning mechanism not yet specified. See TODOS.md item 11.

### Delivery model

- `ipc.send` messages require an `ipc.ack` or `ipc.error` response
- The sender's `send()` promise resolves on `ipc.ack`, throws on `ipc.error`
- No ordering guarantee across concurrent sends
- Messages in flight when the channel drops are not retried — they error
- `ipc.broadcast` and `ipc.sendTo` are fire-and-forget from the worker side

### Message envelope

```json
{
  "feapp_wire": "0.1",
  "type": "...",
  "correlation_id": "uuid",
  "session_id": "uuid | null",
  "event": "event-name",
  "fn": "function-name",
  "payload": {}
}
```

**Fields:**

`feapp_wire` — wire contract version. Both sides must agree before exchanging messages.

`type` — one of the message types below.

`correlation_id` — UUID. Matches `ipc.ack`, `ipc.error`, `stateless.result`, and `stateless.error` back to their originating message. Internal to the runtime — the developer never reads or sets this.

`session_id` — UUID assigned by the runner when a frontend session is established. Stable for the lifetime of that connection. Used to route `ipc.sendTo` and `event.reply()`. Null for external stateless invocations (webhooks).

`event` — event name for IPC messages. Not used for stateless messages.

`fn` — function name for stateless messages. Not used for IPC messages.

`payload` — the data being sent. Must be JSON-serializable.

### Message types

```
ipc.send            frontend → stateful worker
                    requires ipc.ack or ipc.error in response

ipc.ack             stateful worker → frontend
                    confirms receipt of ipc.send
                    carries same correlation_id as the send

ipc.error           stateful worker → frontend
                    delivery failed
                    carries same correlation_id as the send

ipc.broadcast       stateful worker → all connected sessions
                    fire-and-forget

ipc.sendTo          stateful worker → one specific session
                    fire-and-forget
                    session_id identifies target

stateless.invoke    frontend → stateless worker
                    requires stateless.result or stateless.error in response

stateless.result    stateless worker → frontend
                    carries same correlation_id as the invoke

stateless.error     stateless worker → frontend
                    function threw or was not found
                    carries same correlation_id as the invoke
```

### Version mismatch behavior

If either side receives a message with an unsupported `feapp_wire` version, it must close the connection immediately and surface a clear error.

---

## B4 — remoteStorage Protocol

`feapp.storage` is backed by the remoteStorage protocol. This spec references the protocol rather than redefining it.

**Protocol:** remoteStorage (IETF draft, draft-dejong-remotestorage)

> **STUB** — The specific draft revision this spec version targets is not yet pinned. This must be resolved before the spec is final. See TODOS.md item 5.

### What feapp.storage maps to

```
feapp.storage.get(path)                 →  GET {endpoint}{namespace}{path}
feapp.storage.set(path, value)          →  PUT {endpoint}{namespace}{path}
feapp.storage.set(..., {throwOnConflict})→ PUT with If-Match: {etag}
feapp.storage.set(..., {throwIfExists}) →  PUT with If-None-Match: *
feapp.storage.delete(path)              →  DELETE {endpoint}{namespace}{path}
feapp.storage.list(path)                →  GET {endpoint}{namespace}{path}/
feapp.storage.watch(path, handler)      →  runner polls or uses server-sent events
                                           mechanism is implementation-defined
```

The namespace prefix is injected by the runner from profile context. The developer writes to `/articles/123`. The runner translates to the fully qualified path before any operation.

### ConflictError / ExistsError mapping

```
HTTP 412 Precondition Failed  →  feapp.storage.ConflictError
HTTP 404 (on If-Match)        →  feapp.storage.ConflictError
HTTP 412 (on If-None-Match)   →  feapp.storage.ExistsError
HTTP 429 Too Many Requests    →  feapp.storage.RateLimitError  (e.retryAfter from Retry-After header)
HTTP 507 Insufficient Storage →  feapp.storage.StorageFullError
server unreachable            →  feapp.storage.OfflineError
```

# Wire Contracts

This document defines the wire contracts that runner implementations must satisfy.
B3 is a runner requirement — both sides of the IPC channel are implemented by the
cloud library, but the contract is defined here so it can be versioned and evolved
as part of the spec. B4 is the one genuine external interop boundary, where
`feapp.storage` meets independently operated remoteStorage servers.

**Exact version match required within B3.** Both sides of the IPC channel must speak the same `feapp_wire_ipc` version. No compatibility assumptions. No best-effort mode. This applies without exception, especially before v1.0.

---

## Contract index

```
B3  Frontend ↔ Stateful Worker    IPC channel
                                  WebSocket transport
                                  message envelope format
                                  delivery acknowledgment
                                  session routing

B4  Any actor ↔ remoteStorage     external protocol
                                  the one genuine interop boundary
                                  not defined here — referenced
```

B1 (runner launch protocol) is internal to each cloud library distribution. Not a protocol contract — not defined here.

B2 (Desktop shell ↔ Cloud library) is defined in the ecosystem spec WIRE.md. Not a `.feapp` format concern.

---

## B3 — Frontend ↔ Stateful Worker IPC Channel

Defines the WebSocket-based IPC channel between the frontend and the stateful worker. Both sides of this channel are implemented by the cloud library — the cloud library injects the frontend runtime and runs the stateful worker. The protocol is prescribed here so it can be versioned and evolved deliberately as the spec develops.

### Transport

The cloud library exposes a WebSocket upgrade endpoint at a well-known path on the app's subdomain-isolated origin:

```
/feapp/stateful/ws
```

The frontend runtime connects to this endpoint at launch. The cloud library injects the full endpoint URL into the frontend runtime — the developer never constructs or handles it directly.

### Session establishment

On WebSocket open, the frontend runtime sends:

```json
{
  "feapp_wire_ipc": "0.1",
  "type": "session.hello",
  "session_id": "uuid"
}
```

`session_id` is assigned by the cloud library and injected into the frontend runtime at launch. The cloud library responds with `session.ready` on success, or closes the connection immediately if the `feapp_wire_ipc` version is not supported.

```json
{
  "feapp_wire_ipc": "0.1",
  "type": "session.ready",
  "session_id": "uuid"
}
```

### Delivery model

- `ipc.send` messages require an `ipc.ack` or `ipc.error` response
- The sender's `send()` promise resolves on `ipc.ack`, throws on `ipc.error`
- No ordering guarantee across concurrent sends
- Messages in flight when the channel drops are not retried — they error
- `ipc.broadcast` and `ipc.sendTo` are fire-and-forget from the worker side

### Message envelope

```json
{
  "feapp_wire_ipc": "0.1",
  "type": "...",
  "correlation_id": "uuid",
  "session_id": "uuid",
  "event": "event-name",
  "fn": "function-name",
  "payload": {}
}
```

**Fields:**

`feapp_wire_ipc` — B3 protocol version. Validated on every message, not only at session establishment. If either side receives a message with an unsupported version, it must close the connection immediately and surface a clear error.

`type` — one of the message types below.

`correlation_id` — UUID. Matches `ipc.ack`, `ipc.error`, `stateless.result`, and `stateless.error` back to their originating message. Internal to the runtime — the developer never reads or sets this.

`session_id` — UUID assigned by the cloud library at session establishment. Stable for the lifetime of that connection. Used to route `ipc.sendTo` and `event.reply()`. Always present — stateless functions are only invoked from frontend sessions.

`event` — event name for IPC messages. Not used for stateless messages.

`fn` — function name for stateless messages. Not used for IPC messages.

`payload` — the data being sent. Must be JSON-serializable.

### Message types

```
session.hello       frontend → cloud library
                    first message on WebSocket open
                    declares feapp_wire_ipc version and session_id

session.ready       cloud library → frontend
                    acknowledges session establishment

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

---

## B4 — remoteStorage Protocol

`feapp.storage` is backed by the remoteStorage protocol. This is the one genuine external interop boundary — remoteStorage servers are independently operated and must conform to the published protocol. This spec references the protocol rather than redefining it.

**Protocol:** `draft-dejong-remotestorage-26`

### What feapp.storage maps to

```
feapp.storage.get(path)                  →  GET {endpoint}{namespace}{path}
feapp.storage.set(path, value)           →  PUT {endpoint}{namespace}{path}
feapp.storage.set(..., {throwOnConflict})→  PUT with If-Match: {etag}
feapp.storage.set(..., {throwIfExists})  →  PUT with If-None-Match: *
feapp.storage.delete(path)               →  DELETE {endpoint}{namespace}{path}
feapp.storage.list(path)                 →  GET {endpoint}{namespace}{path}/
feapp.storage.watch(path, handler)       →  runner polls or uses server-sent events
                                            mechanism is implementation-defined
```

The namespace prefix is injected by the runner from profile context. The developer writes to `/articles/123`. The runner translates to the fully qualified path before any operation.

### feapp.storage.quota()

`quota()` has no mapping in `draft-dejong-remotestorage-26` — the protocol defines no quota query endpoint. The cloud library satisfies `quota()` through a sidecar extension it implements on top of remoteStorage. The sidecar is not part of the remoteStorage protocol and is not defined here — see ecosystem spec ARCHITECTURE.md § Storage Quota Extension.

For external remoteStorage servers that do not implement the sidecar, `quota()` returns `{ used: -1, available: -1 }`. This is not an error.

### Error mapping

```
HTTP 412 Precondition Failed  →  feapp.storage.ConflictError
HTTP 404 (on If-Match)        →  feapp.storage.ConflictError
HTTP 412 (on If-None-Match)   →  feapp.storage.ExistsError
HTTP 429 Too Many Requests    →  feapp.storage.RateLimitError  (e.retryAfter from Retry-After header)
HTTP 507 Insufficient Storage →  feapp.storage.StorageFullError
server unreachable / timeout  →  feapp.storage.OfflineError
unexpected response           →  feapp.storage.ServerError     (e.status: HTTP code or null)
```

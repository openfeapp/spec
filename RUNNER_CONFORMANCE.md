# Runner Conformance

A runner is a stateless executor. It receives a `.feapp` file and profile context, and executes the app. It owns no persistent state.

```
(profile context + .feapp file) → running app
```

This document is the implementor's view of API.md. The behavioral contracts are defined in API.md. This document specifies what a runner must do to satisfy those contracts.

---

## Opening a .feapp file

- Parse the `.feapp` ZIP archive and read `feapp.json`
- Validate the manifest against the declared spec version
- If the spec version is not supported: refuse, tell the user clearly which version is required and how to obtain a compatible runner. No best-effort mode. No partial support.
- If the manifest is malformed or missing required fields: refuse with a clear error

---

## Presenting permissions

The library presents all permissions declared in the manifest to the user as a checklist on first run. The user approves or rejects each permission individually. The library stores these decisions.

On subsequent runs, the library re-presents only if the manifest declares permissions that are not in the stored decisions (e.g., after a version upgrade).

The runner does not present permissions. The runner receives the granted set from the library and enforces it.

---

## Enforcing permissions

The runner receives the granted permission set from the library at launch via the B1 profile context. It enforces exactly this set — nothing more, nothing less. If a permission cannot be enforced, refuse to run the component that requires it. Never silently allow.

Specific enforcements:

```
Frontend network:         block all requests to addresses not in granted frontend_network
                          enforce via Content-Security-Policy
                          frontend_network does not include network_configured endpoints
Worker network:           merge granted worker_network and network_configured values
                          block all requests to addresses not in the merged set
                          same enforcement for both stateful and stateless workers
                          includes WebSocket outgoing connections
Storage paths:            block feapp.storage access outside declared storage.paths
```

---

## Launching the frontend

- Launch in a sandboxed webview
- Inject the `feapp.*` runtime into the frontend at launch via the profile context
- Enforce the manifest network allowlist at the webview level

The frontend must be iframe-compatible for cloud runner embedding:
- Must not set `X-Frame-Options: DENY` or `SAMEORIGIN`
- Must not contain framebusting scripts
- `Content-Security-Policy frame-ancestors` must permit the runner origin

---

## Launching workers

- Launch workers as sandboxed TC55 processes
- Guarantee storage is connected before the stateful worker module begins executing
- If storage cannot be connected: refuse to start the worker, surface the error — do not start the worker in a degraded state
- Expose the stateful worker's IPC endpoint to the frontend via the injected runtime

**Worker instance model:**

```
Stateful:   exactly one instance per profile
            never two, never zero while the app is active (unless disabled)
            lifecycle is managed by the library, not the runner

Stateless:  one fresh module context per invocation
            no state carries over between invocations
            the runner may pre-load the module for performance
            but must clear module-level state before each invocation
```

A runner that cannot provide a persistent stateful worker process must refuse to open any `.feapp` file that declares `workers.stateful`. This is not a partial support scenario — the same refusal behavior as an unsupported spec version. The runner must tell the user clearly that stateful worker support is not available in this runner.

---

## Worker environment guarantees

The worker execution environment must expose:
- TC55 Minimum Common API (the snapshot declared in `tc55` manifest field)
- `feapp.*` worker APIs as defined in API.md
- **WebSocket client API (WHATWG)** — guaranteed in the stateful worker environment. The worker can open outgoing WebSocket connections to any address in the granted network set.
- **WebAssembly** — guaranteed in both stateful and stateless worker environments. WASM execution is subject to the same healthcheck as JavaScript.
- **Console API** — `console.log`, `console.warn`, `console.error` are captured by the runner and surfaced alongside `feapp.log` output in library UI. They must not be silently discarded.

Must block: browser globals (`document`, `window`, DOM APIs), Node.js globals (`process`, `require`, `Buffer`), Deno globals, any runtime-specific extension not in TC55.

Enforcement mechanism is implementation-defined (V8 isolate, prelude script, custom runtime). The outcome is not: a worker must not be able to access globals outside the above set. If enforcement cannot be guaranteed, refuse to run the worker.

---

## Healthcheck

The runner must implement a periodic healthcheck for both worker types to detect a blocked event loop.

```
Ping interval:    every 1 second
Kill threshold:   10 seconds (configurable in library, default 10s)
Mechanism:        runner pings worker on internal control channel
                  runtime responds automatically — not developer-facing
                  developer code never sees the ping
```

**On stateful worker healthcheck failure:** kill the worker process, initiate restart with backoff (see Restart Policy).

**On stateless worker healthcheck failure:** kill the specific invocation, return `feapp.stateless.ExecutionError` to the caller.

---

## Restart Policy

When the stateful worker is killed by the healthcheck or crashes unexpectedly, the runner must restart it with the following backoff sequence:

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

This is a hard requirement. The exact sequence must be followed — runners must not use a different backoff schedule.

**On disable:** Set `feapp.worker.stateful.status` to `'disabled'`. The frontend can still open, read/write `feapp.storage`, and invoke stateless functions. The library must surface a notification to the user. The frontend cannot launch from the library while disabled. If the frontend is already open when disable triggers, the status change propagates to the frontend via `feapp.worker.stateful.on('status-change')`.

**On re-enable:** The user re-enables from library settings. The retry counter resets to zero. The runner immediately attempts to start the worker. If it fails, the backoff sequence starts over from attempt 1.

**On successful start:** A successful start is defined as the worker responding to healthcheck pings consistently (surviving at least one healthcheck cycle after `worker-start` fires). A successful start resets the retry counter to zero.

---

## Stateless invocation timeout

Each stateless invocation has a timeout. The default is **1 minute**. The timeout is configurable in the library.

If the function does not return within the timeout, the runner kills the invocation and returns a `feapp.stateless.TimeoutError` to the caller. This protects against hung async operations that do not block the event loop (and therefore are not caught by the healthcheck).

---

## TC55 enforcement

The worker execution environment must expose only:
- TC55 Minimum Common API (the snapshot declared in `tc55` manifest field)
- `feapp.*` worker APIs as defined in API.md
- WebSocket client API (stateful worker only)
- WebAssembly
- Console API

Must block: browser globals (`document`, `window`, DOM APIs), Node.js globals (`process`, `require`, `Buffer`), Deno globals, any runtime-specific extension not in TC55.

Enforcement mechanism is implementation-defined (V8 isolate, prelude script, custom runtime). The outcome is not: a worker must not be able to access globals outside the above set. If enforcement cannot be guaranteed, refuse to run the worker.

---

## Browser storage namespacing

The runner wraps all browser storage APIs and enforces namespacing by app ID and profile. The developer uses browser storage normally. All keys, store names, database names, and cookie names are prefixed transparently.

```
Developer writes:   localStorage.setItem('theme', 'dark')
Runner stores as:   localStorage.setItem('{app_id}:{profile}:theme', 'dark')
```

On every profile switch, clear all namespaced browser storage for the previous profile before launching the new profile. This applies to: IndexedDB, localStorage, sessionStorage, cookies, Cache API, OPFS.

Apps must not be able to access each other's browser storage. This is enforced by namespacing, not convention.

---

## Profile isolation

Profile isolation is a hard structural guarantee. Enforce at every level:

```
Storage paths:    namespaced by profile — cross-profile access is structurally impossible
IPC channels:     one channel per profile, separately authenticated
Browser storage:  namespaced by profile — cross-profile local access is impossible
```

A runner that cannot enforce profile isolation must refuse to run multiple profiles simultaneously.

---

## Profile context

The runner receives profile context from the library via the B1 wire contract (see WIRE.md). The runner uses this context to configure storage namespacing, worker endpoints, and runtime injection. The worker never sees raw profile context.

The runner must not persist profile context beyond the current session.

---

## Storage authority

The stateful worker must always run on the same host as the remoteStorage server it writes to. This prevents split-brain writes from multiple worker instances.

If the active remoteStorage server is external (not local), the runner must not start a local stateful worker. Refuse with a clear error.

---

## Version compatibility

The runner must declare which spec versions it supports. Per version:

```
v0.x    Each minor version may break. A v0.1 runner must not run v0.2 apps.
        Each minor version is treated as independent.

v1.0+   Full backward compatibility required within a major version.
        A v1.5 runner must run all v1.x apps.
        Cross-major compatibility is not guaranteed.
```

On version mismatch: tell the user which version is required. Do not attempt to run in a degraded or best-effort mode.

---

## Connectivity diagnostic

The runner must expose a diagnostic endpoint callable by the library:

- Tests each active connection point for the current profile
- Reports: reachable/unreachable, latency, auth status, read/write capability
- Uses the remoteStorage ETag mechanism for read/write tests
- Results are surfaced in library UI only — never inside a `.feapp` frontend

---

## What a conformant runner must not do

```
Run a worker it cannot enforce TC55 compliance for
Run a worker it cannot enforce manifest permissions for
Allow cross-profile communication or storage access
Run a stateful worker when storage authority belongs to another host
Grant permissions the user has not approved — the runner enforces only what the library passes
Attempt to run an app whose spec version it does not support
Persist profile context beyond the current session
Start a stateful worker before storage is confirmed connected
Allow module-level state to leak between stateless invocations
Use a restart backoff schedule other than the one specified
Silently discard console.log output from workers
```

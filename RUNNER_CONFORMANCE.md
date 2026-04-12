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

Before first run, present the user with all permissions declared in the manifest:
- Network hosts the frontend may contact
- Network hosts each worker may contact
- Filesystem paths the stateful worker may access
- Optional external services
- Storage paths the app uses

On subsequent runs, re-present only if permissions have changed since the user last approved.

---

## Enforcing permissions

Enforce all declared permissions structurally — not by convention. If a permission cannot be enforced, refuse to run the component that requires it. Never silently allow.

Specific enforcements:

```
Frontend network:         block all requests to hosts not in frontend.network.allowed
                          block localhost access except declared ports
Stateful worker network:  block all requests to hosts not in workers.stateful.permissions.network
Stateless worker network: block all requests to hosts not in workers.stateless.permissions.network
Storage paths:            block feapp.storage access outside declared storage.paths
Filesystem paths:         block feapp.fs access outside declared filesystem permissions
Stateless storage:        feapp.storage.set(), delete(), and watch() must throw PermissionError
                          feapp.fs must not be available to stateless workers
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
            never two, never zero while the app is active
            lifecycle is managed by the library, not the runner

Stateless:  one instance per concurrent invocation
            runner manages warm/cold lifecycle as implementation detail
            must always run locally next to the runner
```

A runner that cannot provide a persistent stateful worker process must refuse to open any `.feapp` file that declares `workers.stateful`. This is not a partial support scenario — the same refusal behavior as an unsupported spec version. The runner must tell the user clearly that stateful worker support is not available in this runner.

---

## TC55 enforcement

The worker execution environment must expose only:
- TC55 Minimum Common API (the snapshot declared in `tc55` manifest field)
- `feapp.*` worker APIs as defined in API.md

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

The runner must declare which spec versions it support. Per version:

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
Allow a stateless worker to write to feapp.storage
Allow a stateless worker to use feapp.fs
Run a stateful worker when storage authority belongs to another host
Silently grant permissions the user has not acknowledged
Attempt to run an app whose spec version it does not support
Persist profile context beyond the current session
Start a stateful worker before storage is confirmed connected
```

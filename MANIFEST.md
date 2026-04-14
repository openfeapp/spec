# Manifest Specification

The manifest is the most important artifact in a `.feapp` file. It is a trust declaration, a time capsule, and a contract between the developer, the user, and any future runner.

Every field exists for a reason. Every field that is not declared does not happen.

---

## Format

A JSON file named `feapp.json` at the root of the `.feapp` archive.

---

## Example Manifests

### Minimal — frontend only

```json
{
  "feapp": "0.1",
  "id": "com.developer.mygame",
  "name": "My Game",
  "version": "1.0.0",
  "author": {
    "name": "Developer Name"
  },
  "frontend": {
    "entry": "index.html",
    "compat_lock": "compat.lock.json",
    "network": {
      "allowed": []
    }
  },
  "storage": {
    "paths": ["/mygame/"]
  },
  "data": {
    "export_format": "json",
    "schema_version": 1
  }
}
```

### With stateful and stateless workers

```json
{
  "feapp": "0.1",
  "id": "com.developer.feedreader",
  "name": "Feed Reader",
  "version": "1.0.0",
  "author": {
    "name": "Developer Name"
  },
  "frontend": {
    "entry": "index.html",
    "compat_lock": "compat.lock.json",
    "network": {
      "allowed": ["https://cdn.example.com"]
    }
  },
  "workers": {
    "permissions": {
      "network": ["https://api.example.com"]
    },
    "stateful": {
      "entry": "workers/stateful.js",
      "tc55": "2025"
    },
    "stateless": {
      "entry": "workers/stateless.js",
      "tc55": "2025"
    }
  },
  "network_configured": [
    {
      "name": "llm",
      "purpose": "Local AI inference",
      "default": "http://localhost:11434",
      "count": 1
    },
    {
      "name": "feeds",
      "purpose": "RSS feed sources"
    }
  ],
  "storage": {
    "paths": ["/feedreader/"]
  },
  "data": {
    "export_format": "json",
    "schema_version": 1
  },
  "distribution": {
    "update_feed": "https://developer.com/feedreader/releases.json",
    "license": "one-time"
  }
}
```

### With external native services

```json
{
  "feapp": "0.1",
  "id": "com.developer.videolibrary",
  "name": "Video Library",
  "version": "1.0.0",
  "author": {
    "name": "Developer Name"
  },
  "frontend": {
    "entry": "index.html",
    "compat_lock": "compat.lock.json",
    "network": {
      "allowed": []
    }
  },
  "workers": {
    "permissions": {
      "network": []
    },
    "stateful": {
      "entry": "workers/stateful.js",
      "tc55": "2025"
    }
  },
  "network_configured": [
    {
      "name": "filesystem",
      "purpose": "Local filesystem watcher for video cataloging",
      "default": "ws://localhost:9876",
      "count": 1
    },
    {
      "name": "transcoder",
      "purpose": "Local video transcoding service",
      "default": "http://localhost:9877",
      "count": 1
    }
  ],
  "storage": {
    "paths": ["/videolibrary/"]
  },
  "data": {
    "export_format": "json",
    "schema_version": 1
  }
}
```

---

## Field Reference

### `feapp`

**Type:** string
**Required:** yes

The spec version this manifest targets. A runner that does not support this exact version must refuse to open the file and tell the user clearly. No best-effort mode. No compatibility assumptions.

---

### `id`

**Type:** string
**Required:** yes
**Format:** reverse domain notation (`com.developer.appname`)

Globally unique identifier for this app. Stable across versions — changing the ID creates a new app as far as any runner or library is concerned.

---

### `name`

**Type:** string
**Required:** yes

Human-readable display name. Used in runner UI, library UI, and available to all actors via `feapp.app.name`.

---

### `version`

**Type:** string
**Required:** yes
**Format:** semantic versioning (`major.minor.patch`)

The version of this `.feapp` file. Used by libraries to compare against update feeds and to enforce exact version matching across components.

---

### `author`

**Type:** object
**Required:** yes

```json
{
  "name": "Human-readable author name"
}
```

`name` is displayed to the user by the runner and library before the app is run.

Signing is not part of this spec. Ecosystems that implement signing extend the `author` object with additional fields (e.g. `signing_key`). See the [ecosystem spec](https://github.com/openfeapp/ecosystem-spec) for the signing model used by the Forever App ecosystem.

---

### `frontend`

#### `frontend.entry`

**Type:** string
**Required:** yes

Path to the entry HTML file inside the `.feapp` archive. Typically `index.html`.

#### `frontend.compat_lock`

**Type:** string
**Required:** yes

Path to a compatibility lockfile inside the `.feapp` archive. The lockfile is a self-describing artifact generated at packaging time. The developer does not write it by hand — the CLI generates it automatically.

The generation workflow:

1. `@openfeapp/web-compat-findings` scans the app's built frontend assets and emits a `compat-findings/v1` artifact. The scanner discovers JS, CSS, and HTML files, extracts usage signals, and maps them to canonical BCD compatibility keys with concrete evidence.
2. `compat-generate-lock` from `@openfeapp/web-compat` combines the scanner findings with the spec's floor requirements (see [Frontend platform floor](#frontend-platform-floor)) to produce a `compat-lock/v1` artifact. App requirements already satisfied by the floor are omitted — the lockfile captures only the delta above the floor.
3. The lockfile is embedded in the `.feapp` archive at the path declared here.

The runner reads the `format` field of the referenced file to determine how to resolve it. If the runner recognizes the format, it resolves against its own webview engine and version and warns the user if any required API is unavailable. If the runner does not recognize the format, it must refuse to launch and tell the user which format version is required.

The feapp spec does not define the lockfile format. The format is owned by its own specification. The current recommended format is `compat-lock/v1` ([openfeapp/web-compat](https://github.com/openfeapp/web-compat)).

```json
"compat_lock": "compat.lock.json"
```

#### `frontend.network`

**Type:** object
**Required:** yes

```json
"network": {
  "allowed": ["https://api.example.com", "ws://realtime.example.com"]
}
```

`allowed` — network addresses the frontend is permitted to contact. Value is either an array of addresses or `"*"` (unrestricted). Empty array means fully offline. See [Network addresses](#network-addresses) for the address syntax.

```json
"network": { "allowed": "*" }
```

The runner enforces frontend network permissions via [Content-Security-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy). The address syntax defined in this spec is designed to be translatable to CSP `connect-src`, `img-src`, and related directives. Runners must produce a valid CSP from the granted frontend network set. When `"*"` is granted, the runner sets the relevant CSP directives to `*`.

The frontend does not have access to `network_configured` endpoints or worker network permissions. Its network access is defined entirely by `frontend.network.allowed`.

---

### `workers`

**Required:** no

Declares stateful and/or stateless workers. Omit entirely for frontend-only apps. Either `stateful` or `stateless` may be omitted if that worker type is not used.

#### `workers.permissions`

**Required:** yes (if workers declared)

Worker permissions are declared once and shared by both the stateful and stateless worker. There is no per-worker permission scoping — both workers operate under the same network set.

```json
"permissions": {
  "network": ["https://feeds.example.com", "http://localhost:11434"]
}
```

`network` — network addresses both workers may contact. Value is either an array of addresses or `"*"` (unrestricted). Empty array means no external network access. `network_configured` endpoints are merged into this set at enforcement time. See [Network addresses](#network-addresses) for the address syntax.

`storage` access is implicit, not declared. All actors always have read-write access to `feapp.storage`. This is structural — not a permission the user grants or the manifest declares.

#### `workers.stateful`

```json
"stateful": {
  "entry": "workers/stateful.js",
  "tc55": "2025"
}
```

`entry` — path to the stateful worker module inside the archive.

`tc55` — the TC55 (WinterTC Minimum Common API) snapshot year the worker was built against. The runner exposes exactly the API surface defined by this snapshot. This is the time-capsule field for workers.

#### `workers.stateless`

```json
"stateless": {
  "entry": "workers/stateless.js",
  "tc55": "2025"
}
```

Same fields as `workers.stateful`.

---

### `network_configured`

**Type:** array of objects
**Required:** no

For network endpoints the developer cannot know at build time — user-chosen servers, local services, self-hosted infrastructure, native capability bridges.

```json
"network_configured": [
  {
    "name": "llm",
    "purpose": "Local AI inference",
    "default": "http://localhost:11434",
    "count": 1
  },
  {
    "name": "feeds",
    "purpose": "RSS feed sources"
  },
  {
    "name": "filesystem",
    "purpose": "Local filesystem watcher",
    "default": "ws://localhost:9876",
    "count": 1
  }
]
```

`network_configured` is declared at the top level of the manifest. Configured endpoints are merged into the worker network enforcement set. They are not available to the frontend.

`name` — required. `a-z0-9` only. Unique within the array. Used as the key in the `feapp.permissions.configured()` API.

`purpose` — required. Human-readable string shown to the user by the library. Explains what this endpoint is for.

`default` — optional. A valid network address. Pre-filled in the library UI. The user can change it.

`count` — optional integer, minimum 1. If present, the library enforces that the user configures at most this many endpoints for this slot. If absent, no limit — the user can add as many as they want. The CLI rejects `count: 0`.

The library presents configured permissions as input fields, not just checkboxes. The user provides endpoint values, or skips the slot entirely. The library stores the configured values.

All configured values are subject to the same network address syntax and validation rules as static network permissions. Wildcards are permitted in user-provided values (e.g., the user can configure `**.wordpress.com` as a feed source).

---

### `storage`

**Required:** no

Declares the `feapp.storage` paths this app uses.

```json
"storage": {
  "paths": ["/myapp/"],
  "reserved_path_override": "/myapp-internal/"
}
```

`paths` — path prefixes the app reads and writes. Must end with `/`.

`reserved_path_override` — remaps the `/feapp/` reserved namespace for this app only. Last resort for apps with existing data at paths under `/feapp/`. CLI warns loudly if present.

---

### `data`

#### `data.export_format`

**Type:** string
**Values:** `json`

The format the app uses for data export. Every `.feapp` app must support data export.

#### `data.schema_version`

**Type:** integer

The current version of the app's data schema. When a new version declares a higher schema version, the runner prompts the user and runs a migration before launching.

---

### `distribution`

**Required:** no

```json
"distribution": {
  "update_feed": "https://developer.com/myapp/releases.json",
  "license": "one-time"
}
```

`update_feed` — URL to a feed the library polls for new versions. The format of the feed is defined by the ecosystem, not this spec.

`license` — `"one-time"`, `"free"`, or `"open-source"`. Informs the library how to present the app.

---

## Frontend Platform Floor

Each spec version defines a frontend platform floor — the minimum web platform surface that every conformant runner must guarantee in its webview environment. The floor is tied to the spec version and never changes after publication. A new spec version may raise the floor but must not shrink it — the floor is monotonic across spec versions. The floor for version N is always a subset of the floor for version N+1.

The floor for feapp is defined by the `compat-requirements/v1` files in `floor/` of the spec repository:

```
floor.language.requirements.json    ES2022 language built-ins
floor.api.requirements.json         Web platform APIs
floor.html.requirements.json        HTML elements
floor.css.requirements.json         CSS features
```

These files are machine-readable and consumable by `@openfeapp/web-compat` tooling. They are the normative definition. The summary below is informational.

### ES2022 language level

The runner's webview must support ES2022 without transpilation. This includes top-level `await`, public and private class fields, private methods, `Array.prototype.at()`, `Object.hasOwn()`, `Error.cause`, and the RegExp `d` flag (`hasIndices`).

### Web platform APIs

Storage and data: IndexedDB, `localStorage`, `sessionStorage`, Cache API, Origin Private File System (OPFS via `navigator.storage.getDirectory()`), `structuredClone`.

Network: `fetch`, `AbortController`, `AbortSignal`, `URL`, `URLSearchParams`, `Headers`, `Request`, `Response`, `FormData`, `WebSocket`.

Streams: `ReadableStream`, `WritableStream`, `TransformStream`.

Encoding: `TextEncoder`, `TextDecoder`.

Crypto: `SubtleCrypto` (via `crypto.subtle`).

Observers: `ResizeObserver`, `IntersectionObserver`, `MutationObserver`.

Scheduling: `queueMicrotask`.

Binary: `Blob`, `File`, `FileReader`.

DOM: `CustomEvent`, `EventTarget`, `CSS.supports()`.

Canvas and media: `HTMLCanvasElement`, `CanvasRenderingContext2D`, `matchMedia`.

Performance: `Performance.now()`, `PerformanceObserver`.

Clipboard: `navigator.clipboard` (Clipboard API).

Animation: Web Animations API (`Element.animate()`, `Animation`).

### HTML elements

`<dialog>`, `<details>`, `<template>`, `<slot>`.

### CSS features

Custom properties (`var()`), Grid, Flexbox `gap`, `aspect-ratio`, `clamp()`, `min()`, `max()`, `:is()`, `:where()`, `scroll-behavior`, logical properties (`margin-inline`, etc.).

### Derived browser floors

The minimum browser versions implied by this floor (informational, derived from BCD):

```
Chrome ≥ 98
Firefox ≥ 111
Safari ≥ 16
```

### How the floor interacts with the lockfile

The feapp CLI automatically includes the spec's floor requirements when generating the lockfile. App requirements already satisfied by the floor are omitted from the lockfile's top-level `requirements` list — the lockfile captures only the delta above the floor. A minimal app that uses nothing beyond the floor still has a lockfile; it simply has an empty `requirements` list.

---

## Permissions

Every permission in the manifest is a request. The user decides what to grant. The app runs regardless — operations on ungrantable resources throw `feapp.PermissionError`.

The manifest is the ceiling. No mechanism — runtime, library, user — can grant a permission not declared in the manifest.

### Permission lifecycle

```
1. Developer declares permissions in the manifest.
2. Library presents all permissions to the user as a checklist on first run.
3. User approves or rejects each permission individually.
4. Library stores the user's decisions.
5. On launch, library passes the granted set to the runner.
6. Runner enforces exactly the granted set. Nothing more, nothing less.
```

On version upgrade, the library diffs the new manifest's permissions against its stored decisions. New permissions are presented to the user. Previously approved permissions remain approved. Previously rejected permissions remain rejected. The user can change any decision from library settings at any time.

### Network addresses

A network address appears in `frontend.network.allowed`, `workers.permissions.network`, and `network_configured` entries. The syntax is:

```
[protocol://]host[:port]
[protocol://]*.host[:port]
[protocol://]**.host[:port]
```

`protocol` — one of `http`, `https`, `ws`, `wss`, or `*`. If `://` is omitted entirely, `*` is assumed (all protocols permitted).

`host` — a valid host as defined by the [WHATWG URL Standard](https://url.spec.whatwg.org/#host-parsing). This includes domain names, IPv4 addresses, and IPv6 addresses.

`port` — optional. If omitted, all ports on that host are permitted. If present, only that specific port is permitted.

`*.` — single-level wildcard. `*.example.com` matches `api.example.com` but does not match `deep.sub.example.com` or `example.com` itself.

`**.` — multi-level wildcard. `**.example.com` matches `api.example.com`, `deep.sub.example.com`, and any depth of subdomain. Does not match `example.com` itself.

**Wildcard depth rule:** After `*.` or `**.` there must be at least two domain segments. `*.example.com` is valid. `*.com` is not. `**.co.uk` is not. `**.example.co.uk` is valid. Wildcards are only valid before domain hosts, not before IP addresses. The CLI rejects invalid wildcards.

**Examples:**

```
api.example.com                  any protocol, any port
https://api.example.com          HTTPS only, any port
http://localhost:11434           HTTP only, port 11434
ws://192.168.1.50:8080           WebSocket only, port 8080
*.example.com                    any protocol, single-level subdomains
**.cdn.example.com:443           any protocol, any subdomain depth, port 443
```

### Frontend vs worker network

Frontend and worker network permissions use the same address syntax but are enforced differently and serve different roles.

```
                        Frontend                    Workers
Enforcement             CSP in the webview          Runner process-level
Configured endpoints    Not included                Included
Scope                   UI-facing requests          Backend operations
Protocols in practice   Mixed content rules apply   No browser restrictions
                        (browser may block HTTP     (HTTP to local services
                        from HTTPS contexts)        works without issue)
```

The frontend network set is for resources the UI itself needs — CDNs, APIs called directly from browser code, WebSocket connections for real-time UI updates.

The worker network set is for backend operations — fetching feeds, calling LLM APIs, syncing to user-configured servers, connecting to local native services over WebSocket. Configured endpoints (`network_configured`) are merged into the worker set only.

### How the runner receives permissions

At launch, the library passes the complete granted set to the runner as part of the B1 profile context:

```json
{
  "granted": {
    "frontend_network": [
      "https://api.example.com",
      "ws://realtime.example.com"
    ],
    "worker_network": [
      "https://feeds.example.com",
      "http://localhost:11434"
    ],
    "network_configured": {
      "llm": ["http://localhost:11434"],
      "feeds": ["https://blog.example.com", "**.wordpress.com"],
      "filesystem": ["ws://localhost:9876"]
    }
  }
}
```

Any `_network` field may be `"*"` instead of an array, meaning unrestricted.

The runner merges `worker_network` and all `network_configured` values into one enforcement set for both workers. At enforcement time, the runner does not distinguish between static and configured origins.

If a permission was declared in the manifest but rejected by the user, it is absent from the granted set. The runner never sees it.

### CLI validation

The CLI validates the manifest at package time:

```
Network addresses match the syntax defined above
Network values are either an array of addresses or "*"
Wildcard depth rule: at least two domain segments after *. or **.
Wildcards only before domain hosts, not IP addresses
network_configured names are a-z0-9, unique
network_configured count is integer >= 1 if present
```

---

## The Manifest as Trust Declaration

Before any part of a `.feapp` runs, the library presents the user with a summary of what the app has declared:

- Network addresses the frontend may contact
- Network addresses the workers may contact
- Configured network endpoints (user provides values)
- Storage paths the app uses

The user approves or rejects each permission individually. The library stores the decisions. The runner enforces the granted set.

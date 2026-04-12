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
    "required_apis": ["indexeddb", "web-crypto", "service-workers"],
    "network": {
      "allowed": [],
      "localhost": [11434]
    }
  },
  "workers": {
    "stateful": {
      "entry": "workers/stateful.js",
      "tc55": "2025",
      "permissions": {
        "network": ["*.rss.com"],
        "storage": "readwrite"
      }
    },
    "stateless": {
      "entry": "workers/stateless.js",
      "tc55": "2025",
      "permissions": {
        "network": ["api.openai.com"],
        "storage": "readonly"
      }
    }
  },
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

### With filesystem access

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
    "network": {
      "allowed": []
    }
  },
  "workers": {
    "stateful": {
      "entry": "workers/stateful.js",
      "tc55": "2025",
      "permissions": {
        "network": [],
        "storage": "readwrite",
        "filesystem": ["/videos", "/thumbnails"]
      }
    }
  },
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

Human-readable display name. Used in runner UI, library UI, and available to the frontend via `feapp.app.name`.

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

#### `frontend.required_apis`

**Type:** array of strings  
**Required:** no  
**Default:** empty array

> **STUB** — The registry or standard used to define valid API identifier strings is not yet resolved. See TODOS.md item 1.

Web platform APIs the frontend requires. The runner checks availability before launching and warns the user if an API is unavailable in the current webview. This is the time-capsule field for the frontend — a runner built in the future uses this to know what environment to provide.

```json
"required_apis": ["indexeddb", "web-crypto", "service-workers"]
```

#### `frontend.network.allowed`

**Type:** array of strings  
**Required:** yes  
**Default:** empty array

Hosts the frontend is permitted to contact. The runner enforces this at the webview level. Empty array means fully offline.

```json
"network": {
  "allowed": ["api.example.com", "*.feeds.io"],
  "localhost": [11434]
}
```

`localhost` declares specific ports the frontend may contact on the local machine. Useful for apps that integrate with local services such as Ollama.

---

### `workers`

**Required:** no

Declares stateful and/or stateless workers. Omit entirely for frontend-only apps. Either key may be omitted if that worker type is not used.

#### `workers.stateful.entry`

**Type:** string  
**Required:** yes (if stateful declared)

Path to the stateful worker module inside the archive.

#### `workers.stateful.tc55`

**Type:** string (year)  
**Required:** yes (if stateful declared)

The TC55 (WinterTC Minimum Common API) snapshot year the worker was built against. The runner exposes exactly the API surface defined by this snapshot. This is the time-capsule field for workers.

```json
"tc55": "2025"
```

#### `workers.stateful.permissions`

**Required:** yes (if stateful declared)

```json
"permissions": {
  "network": ["api.example.com"],
  "storage": "readwrite",
  "filesystem": ["/videos", "/documents"]
}
```

`network` — hosts the worker may contact. Empty array means no external network.

`storage` — `"readwrite"` for stateful workers.

`filesystem` — abstract paths the worker may access via `feapp.fs`. Omit if not needed. The runner maps these to real filesystem paths at launch. The worker never sees real paths.

#### `workers.stateless.entry`

**Type:** string  
**Required:** yes (if stateless declared)

#### `workers.stateless.tc55`

Same as `workers.stateful.tc55`.

#### `workers.stateless.permissions`

```json
"permissions": {
  "network": ["api.openai.com"],
  "storage": "readonly"
}
```

`storage` must be `"readonly"` for stateless workers. Declaring `"readwrite"` is an error — the CLI rejects it and the runner refuses to launch. `filesystem` is not available to stateless workers and is ignored with a warning if declared.

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

### `optional_dependencies`

External services the app can use but does not require. The runner checks availability on launch and informs the user. The app must degrade gracefully if unavailable.

```json
"optional_dependencies": {
  "ollama": {
    "endpoint": "http://localhost:11434",
    "required": false,
    "purpose": "Human-readable explanation shown to user",
    "models": ["llama3.2"]
  }
}
```

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

## The Manifest as Trust Declaration

Before any part of a `.feapp` runs, the runner presents the user with a summary of what the app has declared:

- Network hosts the frontend may contact
- Network hosts each worker may contact
- Filesystem paths the stateful worker may access
- Optional external services
- Storage paths the app uses

The user approves once. The runner enforces permanently.

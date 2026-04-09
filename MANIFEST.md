# Manifest Specification

The `.feapp` manifest is the most important artifact in the ForeverApps ecosystem. It is not just a technical configuration file — it is a **trust declaration**, a **time capsule**, and a **contract** between the developer, the user, and any future runner that needs to execute the app.

Every field exists for a reason. Every field that isn't declared doesn't happen.

---

## Format

The manifest is a JSON file named `feapp.json`, located at the root of the `.feapp` package.

---

## Example Manifests

### Tier 1 — HTML5 Game (frontend only)

```json
{
  "feapp": "0.1",
  "id": "com.developer.mygame",
  "name": "My Game",
  "version": "1.0.0",
  "author": {
    "name": "Developer Name",
    "signing_key": "sha256:abc123..."
  },
  "runtime": {
    "web": {
      "min_chrome": 110,
      "min_firefox": 109,
      "min_safari": 16
    }
  },
  "frontend": {
    "entry": "index.html",
    "network": {
      "allowed": []
    }
  },
  "data": {
    "storage": "indexeddb",
    "export_format": "json",
    "schema_version": 1
  },
  "distribution": {
    "update_feed": "https://developer.com/mygame/releases.json",
    "license": "one-time"
  }
}
```

### Tier 1 — Productivity PWA (frontend only, with network access)

```json
{
  "feapp": "0.1",
  "id": "com.developer.myjournal",
  "name": "My Journal",
  "version": "1.0.0",
  "author": {
    "name": "Developer Name",
    "signing_key": "sha256:abc123..."
  },
  "runtime": {
    "web": {
      "min_chrome": 110,
      "min_firefox": 109,
      "min_safari": 16
    }
  },
  "frontend": {
    "entry": "index.html",
    "network": {
      "allowed": []
    }
  },
  "data": {
    "storage": "indexeddb",
    "export_format": "json",
    "schema_version": 1,
    "remotestorage": {
      "module": "journal",
      "paths": ["/journal/entries/", "/journal/settings/"]
    }
  },
  "distribution": {
    "update_feed": "https://developer.com/myjournal/releases.json",
    "license": "one-time"
  }
}
```

### Tier 2 — App with Deno Background Worker

```json
{
  "feapp": "0.1",
  "id": "com.developer.feedreader",
  "name": "Feed Reader",
  "version": "1.0.0",
  "author": {
    "name": "Developer Name",
    "signing_key": "sha256:abc123..."
  },
  "runtime": {
    "web": {
      "min_chrome": 110,
      "min_firefox": 109,
      "min_safari": 16
    },
    "worker": {
      "runtime": "deno",
      "min_version": "1.40"
    }
  },
  "frontend": {
    "entry": "index.html",
    "network": {
      "allowed": []
    }
  },
  "worker": {
    "entry": "worker/main.ts",
    "permissions": {
      "network": {
        "allowed": ["feeds.example.com", "*.rss.com"]
      },
      "filesystem": {
        "allowed": []
      }
    },
    "communication": {
      "direct": true,
      "remotestorage": true
    },
    "lifecycle": {
      "start": "on_app_open",
      "stop": "on_app_close"
    }
  },
  "data": {
    "storage": "indexeddb",
    "export_format": "json",
    "schema_version": 1,
    "remotestorage": {
      "module": "feedreader",
      "paths": ["/feedreader/feeds/", "/feedreader/articles/", "/feedreader/settings/"]
    }
  },
  "optional_dependencies": {
    "ollama": {
      "endpoint": "http://localhost:11434",
      "required": false,
      "purpose": "Article summarization",
      "models": ["llama3.2"]
    }
  },
  "distribution": {
    "update_feed": "https://developer.com/feedreader/releases.json",
    "license": "one-time"
  }
}
```

---

## Field Reference

### `feapp`
**Type:** string  
**Required:** yes

The spec version this manifest targets. Used by runners to determine compatibility. A runner that does not support this spec version must refuse to open the file and tell the user clearly why.

---

### `id`
**Type:** string  
**Required:** yes  
**Format:** reverse domain notation (`com.developer.appname`)

A globally unique identifier for this app. Used by the runner and library to distinguish apps from each other, manage data namespacing, and prevent conflicts.

---

### `name`
**Type:** string  
**Required:** yes

The human-readable display name of the app.

---

### `version`
**Type:** string  
**Required:** yes  
**Format:** semantic versioning (`major.minor.patch`)

The version of this specific `.feapp` file. The library uses this to compare against the update feed.

---

### `author`
**Type:** object  
**Required:** yes

```json
{
  "name": "Human-readable author name",
  "signing_key": "sha256:fingerprint of the developer's public key"
}
```

The signing key fingerprint links this manifest to the author's key registered with a signing authority. See `SIGNING.md` for the full trust model.

---

### `runtime`

Declares what runtime versions the app was built for and verified against. This is the **time capsule field** — a runner built in the future uses this to know what environment to emulate.

#### `runtime.web`
**Required:** yes

```json
{
  "min_chrome": 110,
  "min_firefox": 109,
  "min_safari": 16
}
```

Minimum browser engine versions the app is known to work with. The runner warns the user if the system webview is older than declared.

#### `runtime.worker`
**Required:** only for Tier 2 apps

```json
{
  "runtime": "deno",
  "min_version": "1.40"
}
```

Supported worker runtimes: `deno`, `docker`. If `docker`, the runner will check for Docker availability and inform the user if it is not installed. Docker is never bundled by the runner — it is a user dependency.

---

### `frontend`

#### `frontend.entry`
**Type:** string  
**Required:** yes

Path to the entry HTML file inside the `.feapp` package. Typically `index.html`.

#### `frontend.network.allowed`
**Type:** array of strings  
**Required:** yes  
**Default:** empty array (no network access)

An explicit list of domains the frontend PWA is permitted to contact. The runner enforces this at the webview level. An empty array means the app is fully offline — it cannot make any external network requests.

```json
"network": {
  "allowed": ["api.openweathermap.org", "*.myservice.com"],
  "localhost": [11434]
}
```

`localhost` ports can be declared separately, for apps that communicate with local services such as Ollama.

---

### `worker`

Only present in Tier 2 apps.

#### `worker.entry`
**Type:** string  
**Required:** yes (Tier 2)

Path to the worker entry file inside the `.feapp` package.

#### `worker.permissions`

Declared permissions the worker requires. The runner presents these to the user before the worker starts for the first time. The user must explicitly approve. The worker cannot exceed what is declared here.

```json
"permissions": {
  "network": {
    "allowed": ["api.example.com"]
  },
  "filesystem": {
    "allowed": ["~/Documents/myapp"]
  }
}
```

#### `worker.communication`

```json
"communication": {
  "direct": true,
  "remotestorage": true
}
```

`direct` — the frontend and worker can communicate directly when both are running on the same device. The mechanism is a local message channel managed by the runner. The worker cannot expose arbitrary ports.

`remotestorage` — the frontend and worker can communicate through remoteStorage. This is the only communication mode available when the worker is running on a different machine (cloud library, Raspberry Pi, VPS).

#### `worker.lifecycle`

```json
"lifecycle": {
  "start": "on_app_open",
  "stop": "on_app_close"
}
```

When the runner starts and stops the worker process. Future values may include `always` (start with runner, run regardless of app state) and `manual` (user controls lifecycle).

---

### `data`

#### `data.storage`
**Type:** string  
**Values:** `indexeddb`

The browser storage mechanism the app uses. Currently only IndexedDB is supported.

#### `data.export_format`
**Type:** string  
**Values:** `json`

The format the app uses for data export. Every ForeverApp must support data export. This is non-negotiable.

#### `data.schema_version`
**Type:** integer

The current version of the app's data schema. Used to manage migrations between app versions. When a new `.feapp` version declares a higher schema version, the runner prompts the user and runs the migration before opening the app.

#### `data.remotestorage`

Optional. Declares the remoteStorage module and paths this app uses.

```json
"remotestorage": {
  "module": "myapp",
  "paths": ["/myapp/data/", "/myapp/settings/"]
}
```

This declaration enables the library to show users which remoteStorage paths each app uses, and enables future tooling to manage cross-app data relationships.

---

### `optional_dependencies`

External services the app can use but does not require. The runner checks for availability on launch and informs the user. The app must degrade gracefully if these are unavailable.

```json
"optional_dependencies": {
  "ollama": {
    "endpoint": "http://localhost:11434",
    "required": false,
    "purpose": "Human-readable explanation shown to user",
    "models": ["llama3.2"]
  },
  "docker": {
    "required": false,
    "purpose": "Heavy background processing"
  }
}
```

---

### `distribution`

#### `distribution.update_feed`
**Type:** string (URL)  
**Required:** no

URL to a JSON feed the library polls to check for new versions. See `UPDATES.md` for the feed format. If absent, the app does not support automatic update notifications.

#### `distribution.license`
**Type:** string  
**Values:** `one-time`, `free`, `open-source`

Informs the library how to present the app to the user.

---

## The Manifest as Trust Declaration

Before any part of a `.feapp` runs, the runner presents the user with a summary of what the app has declared:

- Network domains the frontend can access
- Network domains the worker can access
- Filesystem paths the worker can read or write
- External services the app connects to
- remoteStorage paths the app uses

The user approves once. The runner enforces permanently. No app can do anything that isn't in its manifest.

This is the mechanism that separates ForeverApps from "potential malware" — full transparency before execution, enforced structurally, not by convention.

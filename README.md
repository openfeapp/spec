# openfeapp/spec

The specification for the `.feapp` format — a self-contained, portable file format for web applications that work forever, on any device, without a server.

NOTICE: work in progress. Always looking for suggestions/contributions. Currently run by one person.

---

## What is a `.feapp` file?

A `.feapp` file is to a web application what an executable is to a desktop program, or a ROM is to a video game.

It is a complete, self-contained artifact. It does not require hosting. It does not require a domain. It does not expire when a company shuts down or a developer stops paying for a server. It can be copied, shared, archived, and passed between people like any other file.

A `.feapp` file contains a web application (HTML, CSS, JavaScript), an optional background worker (Deno), and a manifest that declares everything the app needs to run — runtime requirements, network permissions, data schema, signing information, and update feed. The manifest is a time capsule: a runner built decades from now can read it and know exactly what environment the app was built for.

The ForeverApps runner opens `.feapp` files on desktop (Windows, macOS, Linux). The ForeverApps cloud library makes them accessible from any browser, including mobile.

---

## Why does this exist?

Read [VISION.md](./VISION.md).

The short version: every developer who builds something useful eventually faces the same trap. The data lives on someone else's server. The infrastructure has a monthly cost. The app store takes 30%. The developer burns out or moves on, and the app disappears.

`.feapp` is an attempt to fix the infrastructure problem structurally, not through better intentions.

---

## Documents

| Document | What it covers |
|---|---|
| [VISION.md](./VISION.md) | Why this exists. The problems it solves. What came before. |
| [MANIFEST.md](./MANIFEST.md) | The manifest schema. Example manifests. Field reference. |
| [SIGNING.md](./SIGNING.md) | Author verification, trust anchors, key revocation. |
| [PERMISSIONS.md](./PERMISSIONS.md) | Network, filesystem, and worker permission model. |
| [UPDATES.md](./UPDATES.md) | Update feed format. Library update behavior. |
| [ROADMAP.md](./ROADMAP.md) | What gets built, in what order, and why. |

---

## Repositories

| Repository | What it is |
|---|---|
| [`openfeapp/spec`](https://github.com/openfeapp/spec) | This. The spec. Start here. |
| [`openfeapp/runner`](https://github.com/openfeapp/runner) | Desktop runner — opens `.feapp` files on Windows, macOS, Linux |
| [`openfeapp/library`](https://github.com/openfeapp/library) | Desktop library — manages, launches, and updates `.feapp` files |
| [`openfeapp/cli`](https://github.com/openfeapp/cli) | Packages a PWA or existing web app into a `.feapp` file |
| [`openfeapp/sdk`](https://github.com/openfeapp/sdk) | Helpers for building feapp-aware PWAs with remoteStorage |
| [`openfeapp/cloud-runner`](https://github.com/openfeapp/cloud-runner) | Server-side PWA and worker execution |
| [`openfeapp/cloud-library`](https://github.com/openfeapp/cloud-library) | Self-hosted web interface for `.feapp` files |

---

## Status

**Current phase: Phase 0 — Spec Complete**

The spec is the first thing. No implementation exists yet. The roadmap is in [ROADMAP.md](./ROADMAP.md).

---

## What a `.feapp` looks like

A minimal manifest for an HTML5 game:

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

No server. No domain dependency. No account. Opens, runs, saves progress locally. Works offline. Works forever.

---

## Core principles

**The file is the product.** A `.feapp` file can be sold, shared, copied, archived, and passed between people. Distribution is not a gatekeeping problem.

**Declare everything, enforce structurally.** Every network connection, every filesystem access, every external dependency is declared in the manifest. The runner enforces it. The app cannot exceed what it declared.

**The frontend always works without the worker.** The background worker is a capability upgrade, never a dependency. If the worker is unavailable, the app still opens and the user's data is still accessible.

**All communication goes through remoteStorage.** The frontend and background worker communicate exclusively through remoteStorage. No direct IPC, no localhost APIs. This means the worker can run anywhere — same machine, Raspberry Pi, VPS — and the frontend doesn't care.

**Piratability is a feature.** A file that can be freely copied cannot be killed by a company shutting down. The durability of `.feapp` is a property of the artifact, not a promise from a company.

---

## Intellectual lineage

ForeverApps did not invent these ideas. It assembles them:

- **Unhosted / remoteStorage** (2011) — got the architecture right, never built the apps
- **Local-First Software — Ink & Switch** (2019) — articulated the philosophy
- **Web Bundles — Google/W3C** (2019) — got the portable format right, stalled for political reasons
- **Firefox OS** (2013) — got the packaged web app right, died with the platform
- **Electron / Tauri** — got the native shell right, without data ownership opinions
- **PWA movement** — got cross-platform distribution right, didn't solve the hosting dependency

The specific combination — shared runner, self-contained artifact, remoteStorage as the sole communication contract, manifest as time capsule, piratability as durability — does not exist elsewhere. That combination is what this is.

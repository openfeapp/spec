# openfeapp/spec

> Work in progress. Always looking for suggestions and contributions.

STATUS: v0.1 spec revisions

---

A `.feapp` file is to a web app what a ROM is to a video game — a complete, self-contained artifact that runs without a server, cannot be killed by a company shutting down, and works identically whether opened today or in thirty years.

Unlike a ROM, a `.feapp` app has the full architecture of a modern application: a frontend, a continuously running backend, on-demand computation, and storage that syncs across devices. All of it is self-contained. All of it is user-owned. None of it requires a hosting bill.

---

## The problem

Every developer who builds something useful faces the same trap. The data lives on someone else's server. The infrastructure has a monthly cost. The developer burns out or moves on, and the app disappears — taking every user's data with it.

The web platform itself is extraordinarily durable. Browsers treat breaking changes as existential threats. Code from the 1990s still runs. The hosting model built on top of the web is not durable at all.

Remove the hosting requirement, and the web becomes the most durable software runtime that has ever existed. That is what `.feapp` does.

For the full argument: [VISION.md](VISION.md).

---

## The model

`.feapp` is not a request/response framework. It is a **local-first actor system packaged as a portable artifact**.

The stateful worker is an actor — a process that runs continuously, owns the user's data, processes messages from the frontend, and does scheduled work on the user's behalf. The frontend connects to it. The stateless worker handles on-demand computation. Storage syncs automatically. The whole thing ships as one file.

This model makes possible things that request/response frameworks cannot do: background work without infrastructure, real-time push without a separate service, offline-first without a sync library, zero ongoing cost to the developer.

For the full explanation: [MODEL.md](MODEL.md).

---

## Where to go next

**I want to understand the vision and philosophy**
→ [VISION.md](VISION.md)

**I want to understand the model — what .feapp fundamentally is**
→ [MODEL.md](MODEL.md)

**I want to build a `.feapp` app**
→ [OVERVIEW.md](OVERVIEW.md) — start here, it will route you to the right documents

**I want to build a runner or ecosystem**
→ [OVERVIEW.md](OVERVIEW.md) — the implementor paths are there

---

## This spec and the ecosystem

This spec defines the `.feapp` format and runtime contract. It is complete on its own — a conformant runner built from this spec can open and run any `.feapp` file.

The [Forever App ecosystem](https://github.com/openfeapp/ecosystem-spec) is one open implementation of the infrastructure around `.feapp`. It defines how apps are deployed, how user accounts work, how signing and updates are handled, and how desktop and cloud libraries interoperate. Reading the ecosystem spec shows `.feapp` in practice.

---

## Core principles

**The file is the product.** A `.feapp` file can be sold, shared, copied, and archived like any other file. Distribution is not a gatekeeping problem.

**Declare everything, enforce structurally.** Every network connection, every filesystem access, every permission is declared in the manifest. The runner enforces it. The app cannot exceed what it declared.

**Exact version match. No exceptions.** Components must agree on exact versions before communicating. A runner that does not support the declared spec version must refuse — not attempt compatibility, not degrade gracefully. This applies without exception, and is especially critical before v1.0 when every minor version may introduce breaking changes.

**Workers are independent of the frontend.** The stateful worker runs continuously regardless of whether any frontend is open. It owns the app's data lifecycle. The frontend connects to the worker, not the other way around.

**`feapp.storage` is the only durable storage.** All browser storage is ephemeral. `feapp.storage`, backed by the remoteStorage protocol, is the source of truth for all persistent data.

**TC55 for workers.** Worker code targets the WinterTC Minimum Common API. No runtime-specific APIs. The same worker code runs in any conformant runner, today and decades from now.

**Bet on institutions, not implementations.** This spec references ECMAScript (TC39), WinterTC (TC55), WHATWG/W3C, and the remoteStorage IETF draft — standards maintained by entities with stronger longevity incentives than any single vendor or product.

---

## Documents

| Document | Audience | What it covers |
| --- | --- | --- |
| [VISION.md](VISION.md) | Everyone | Why this exists. The problem it solves. |
| [MODEL.md](MODEL.md) | Evaluators, Developers | The model of computation. What .feapp fundamentally is. |
| [OVERVIEW.md](OVERVIEW.md) | All | Navigation by role. |
| [MANIFEST.md](MANIFEST.md) | Developers | Manifest schema and field reference. |
| [API.md](API.md) | Developers, Runner implementors | Complete `feapp.*` API reference. |
| [RUNNER_CONFORMANCE.md](RUNNER_CONFORMANCE.md) | Runner implementors | What a conformant runner must do. |
| [WIRE.md](WIRE.md) | Implementors | Wire contracts B1, B3, B4. |

---

## Intellectual lineage

Forever Apps did not invent these ideas. It assembles them:

- **Unhosted / remoteStorage** (2011) — got the architecture right, never built the apps
- **Local-First Software — Ink & Switch** (2019) — articulated the philosophy
- **Web Bundles — Google/W3C** (2019) — got the portable format right, stalled for political reasons
- **Firefox OS** (2013) — got the packaged web app right, died with the platform
- **Electron / Tauri** — got the native shell right, without data ownership opinions
- **PWA movement** — got cross-platform distribution right, didn't solve the hosting dependency

The specific combination — shared runner, self-contained artifact, local-first actor model, manifest as time capsule, piratability as durability — does not exist elsewhere.

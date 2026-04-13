# Overview

Start here. This document tells you which parts of the spec are relevant to your role.

---

## The three parts of a `.feapp` file

```
feapp.json      manifest — declares what the app needs
frontend/       HTML, CSS, JavaScript — runs in a sandboxed webview
workers/        TC55 JavaScript modules — stateful and/or stateless
```

The runner reads the manifest and constructs the environment. The developer writes against the `feapp.*` API. The runner enforces everything.

---

## If you are evaluating whether `.feapp` is right for your project

Start here:

```
MODEL.md        what .feapp fundamentally is — the model of computation
                read this first. everything else will make more sense after.
MANIFEST.md     what a .feapp file looks like in practice
API.md          what your code can do
```

## If you are building a `.feapp` app

You need:

```
MANIFEST.md             what to put in feapp.json
API.md                  the feapp.* API your code runs against
```

You do not need to read RUNNER_CONFORMANCE.md or WIRE.md. Those are for people building runners and libraries.

To see how `.feapp` apps are deployed and used in practice — across desktop, cloud, and mobile — read the [ecosystem spec](https://github.com/openfeapp/ecosystem-spec).

---

## If you are building a runner

A runner is a process that takes a `.feapp` file and profile context and executes it. You need:

```
MANIFEST.md             what fields you must parse and validate
API.md                  what feapp.* behavior you must provide
RUNNER_CONFORMANCE.md   the complete checklist of runner requirements
WIRE.md                 the wire contracts your runner must implement (B1, B3, B4)
```

---

## If you are building an ecosystem

An ecosystem provides the infrastructure around `.feapp`: libraries, signing, cloud runners, authentication, profile management.

Start with this spec to understand the format and runtime contract. Then read the [ecosystem spec](https://github.com/openfeapp/ecosystem-spec), which defines the wire contract between desktop and cloud libraries (B2) and all other ecosystem-level protocols.

---

## Component map

```
                    ┌─────────────────────────────┐
                    │         .feapp file          │
                    │  feapp.json  frontend/  workers/ │
                    └─────────────────────────────┘
                                   │
                            reads manifest
                                   │
                    ┌──────────────▼──────────────┐
                    │            Runner            │
                    │  enforces permissions        │
                    │  injects feapp.* runtime     │
                    │  manages worker processes    │
                    │  healthcheck enforcement     │
                    └──────────┬──────────┬────────┘
                               │          │
              ┌────────────────▼─┐      ┌─▼──────────────────┐
              │    Frontend       │      │   Stateful Worker   │
              │  feapp.storage    │◄────►│   feapp.storage     │
              │  feapp.ipc        │      │   feapp.ipc         │
              │  feapp.stateless  │      │   feapp.schedule    │
              │  feapp.worker     │      │   feapp.sessions    │
              │  feapp.permissions│      │   feapp.permissions │
              │  feapp.app        │      │   feapp.app         │
              └──────────────────┘      │   feapp.log         │
                        │               │   WebSocket client  │
              ┌─────────▼──────────┐    │   WASM              │
              │  Stateless Worker  │    └─────────────────────┘
              │  feapp.storage     │
              │  feapp.permissions │
              │  feapp.app         │
              │  feapp.log         │
              │  WASM              │
              └────────────────────┘
```

---

## Version compatibility

Before v1.0, every minor version of the spec may introduce breaking changes. Components must declare exact spec version support. A runner must refuse to open a `.feapp` file whose spec version it does not support — no exceptions, no best-effort mode.

At v1.0 and above, full backward compatibility is required within a major version. Cross-major compatibility is not guaranteed and will be defined when v2.0 is planned.

See WIRE.md for how wire contract versions interact with spec versions.

---

## Full architecture narrative

A document walking through the complete architecture end-to-end — format, ecosystem, deployment, developer experience — is planned for when both this spec and the ecosystem spec are stable. It will be linked here.

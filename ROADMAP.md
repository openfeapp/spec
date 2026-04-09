# Roadmap

## Principles

**Spec before code.** Every implementation decision derives from the spec. Nothing gets built that contradicts a decision already made in writing.

**Prove before build.** Each phase proves one thing before the next phase begins. A failed proof changes the plan. A skipped proof creates technical debt that compounds.

**The format is the product.** The runner, the library, the CLI — these are implementations of the format. The format outlasts any implementation. Build the format right.

**Developer adoption first.** Users follow developers. A developer who can go from idea to `.feapp` in a weekend is the growth mechanism. Everything is optimized for that developer's experience before anything else.

---

## Phase 0 — Spec Complete
**Deliverable:** `openfeapp/spec` repository, all documents written and reviewed.

This phase is done when:
- [ ] `VISION.md` — complete
- [ ] `MANIFEST.md` — complete, three example manifests written for: HTML5 game, productivity PWA, Tier 2 app with Deno worker
- [ ] `SIGNING.md` — complete
- [ ] `PERMISSIONS.md` — complete
- [ ] `UPDATES.md` — complete
- [ ] `README.md` — complete
- [ ] Every field in the manifest has a reason. No field exists "for future use."
- [ ] Every open question in the spec is explicitly marked as open, not papered over.

**Gate:** Can you write a complete `feapp.json` for three different app types from memory, without looking at the spec? If yes, the spec is clear enough to implement against.

---

## Phase 1 — Prove the Container
**Deliverable:** A real `.feapp` file that opens and runs.

Take an existing HTML5 game — small, self-contained, already working. Package it as a `.feapp` file using a manual process (no CLI yet). Build the simplest possible runner: a Tauri shell that reads a `.feapp` file, extracts the PWA, and displays it in a webview.

This proves:
- The file format works as a container
- An existing web app can become a `.feapp` without modification
- The runner architecture is sound

The runner at this stage has no library UI, no signing verification, no permission enforcement, no worker support. It is a file opener. Nothing more.

**Why this first:** A playable HTML5 game opening from a `.feapp` file is the clearest possible demonstration of the concept. Anyone who sees it immediately understands what `.feapp` is. This becomes the first demo.

**Gate:** A `.feapp` file, double-clicked on macOS, Windows, and Linux, opens the game and plays correctly. Data saved in the game persists between sessions in IndexedDB.

**Repositories touched:** `runner` (initial), `spec` (corrections from implementation experience)

---

## Phase 2 — Build the CLI
**Deliverable:** `feapp pack` command that produces a valid `.feapp` file.

```bash
feapp pack ./mygame
# → mygame.feapp

feapp validate ./mygame.feapp
# → Manifest valid. Signature: unsigned.

feapp sign ./mygame.feapp --key ./developer.key
# → mygame.feapp signed. Fingerprint: sha256:abc123
```

A developer should be able to take any existing PWA or HTML5 project and produce a `.feapp` file in under five minutes. That is the success criterion for the CLI.

This phase also proves the conversion path that makes the HTML5 games use case real. Thousands of existing HTML5 games can become `.feapp` files with no code changes.

**Gate:** An existing open-source HTML5 game, converted to `.feapp` by someone who has never used the CLI before, following only the README, in under 10 minutes.

**Repositories touched:** `cli` (initial)

---

## Phase 3 — Prove the Worker
**Deliverable:** A Tier 2 `.feapp` that uses a Deno background worker.

Build a simple app that actually requires a background worker — a feed aggregator is the reference implementation. The worker polls RSS/Atom feeds on a schedule and writes articles to remoteStorage. The PWA reads from remoteStorage and displays them.

This proves:
- The two-channel communication architecture works (direct + remoteStorage)
- Deno worker sandboxing is enforceable from the runner
- The permission presentation UI is understandable to users
- remoteStorage as IPC is viable in practice

**The hardest question this phase must answer:** Is remoteStorage fast enough and reliable enough to use as a real-time-ish communication channel between the frontend and worker? What are the actual latency characteristics? What happens during conflict resolution when both sides write simultaneously?

If remoteStorage fails as IPC, this is a fundamental architecture problem that must be resolved before Phase 4.

**Gate:** The feed reader app, running from a `.feapp` file, polls feeds in the background while the PWA is open, updates the display without a page refresh, and continues polling for 30 minutes without errors. Worker permissions are enforced — the worker cannot contact domains not in its manifest.

**Repositories touched:** `runner` (worker support), `sdk` (initial — remoteStorage helpers)

---

## Phase 4 — Build the Library
**Deliverable:** The desktop library application.

The library is the management layer for `.feapp` files. It is itself a desktop application — possibly itself a `.feapp`.

Minimum features:
- Installed apps list with name, version, author
- Install a new app by dropping a `.feapp` file
- Launch apps
- Permission summary displayed before first launch
- Signing verification status displayed per app
- Update notifications when update feeds have new versions
- Manual update confirmation flow
- Version history — old versions kept, user switches explicitly

The library is the thing that makes ForeverApps feel like a platform. Before Phase 4, ForeverApps is a file format. After Phase 4, it is a place.

**Gate:** A user with no prior knowledge of ForeverApps can install the library, drag in three `.feapp` files, see their permissions, launch them, and receive an update notification — following only the library's own UI, with no external documentation.

**Repositories touched:** `library` (initial), `runner` (library integration)

---

## Phase 5 — Cloud Library (Alpha)
**Deliverable:** A self-hostable web interface for `.feapp` files.

The cloud library makes ForeverApps work on mobile. iOS and Android users cannot install the desktop runner, but they can access a self-hosted cloud library through their browser.

Minimum features:
- Upload a `.feapp` file
- Browser-based PWA extraction and serving
- Deno worker execution server-side
- remoteStorage integration (same account as desktop)
- Basic auth for the self-hosted instance

The cloud library is the hardest component to build correctly because it introduces server-side execution of worker code. Docker workers are explicitly not supported in Phase 5 — only Deno workers. The attack surface of running arbitrary Docker containers in a shared cloud environment is too large for an early implementation.

**Gate:** A `.feapp` file uploaded to a self-hosted cloud library instance is accessible from an iPhone browser, works offline after the first load, and syncs data with the desktop runner through remoteStorage.

**Repositories touched:** `cloud-runner` (initial), `cloud-library` (initial)

---

## Phase 6 — First Real App
**Deliverable:** A `.feapp` that real users choose over the subscription alternative.

Only after Phases 0–5 is there a complete platform to build on. Phase 6 is the first app built for end users rather than for proving the platform.

The app is chosen based on what was learned in earlier phases — from developer feedback, from the kinds of apps people tried to convert in Phase 2, from the gaps the CLI and SDK revealed.

The app ships as a `.feapp` from day one. It is the proof that the platform works end-to-end for a real product.

**Success criterion:** A stranger, with no prior knowledge of ForeverApps, installs the app, uses it for a week, and would recommend it to someone else. Not because of the philosophy. Because it is a good app.

---

## What Is Deliberately Deferred

**The signing authority service** — Phase 0–3 use unsigned apps. The signing authority is built in parallel with Phase 4 but is not a gate for earlier phases.

**The public directory** — there is no point building a directory before there are 10+ apps worth listing. The directory is a Phase 6+ outcome.

**Framework extraction** — the SDK grows from real usage patterns. It is not designed upfront.

**Community governance** — premature. The spec is owned by the founder until there are enough contributors to justify a governance model.

**Any organizational or legal structure** — deferred until there is something worth protecting.

---

## Open Questions

These are questions the roadmap cannot answer yet. Each must be resolved in the phase where it becomes a gate:

1. **remoteStorage as IPC** — is it fast enough for real-time-ish communication? (Phase 3)
2. **Direct worker communication channel** — what is the exact mechanism? WebSocket to a runner-managed local port? Named pipe? (Phase 3)
3. **Docker worker cloud support** — can Docker workers ever run safely in the cloud library? (Post-Phase 5)
4. **iOS PWA data eviction** — how does the cloud library handle Safari's IndexedDB eviction behavior? (Phase 5)
5. **Background worker on mobile** — can a Deno worker run persistently in the cloud library when the user's device is asleep? (Phase 5)

# Vision

## The Problem

Every developer who wants to build something faces the same trap.

You have an idea. A tool, a game, a productivity app, something niche and useful. You start building. Then the questions arrive: where does the data live? How do I handle sync? Do I use Firebase? MongoDB Atlas? A managed server? Each choice feels pragmatic in the moment. Each choice adds a liability you didn't sign up for.

Firebase could change its pricing. MongoDB Atlas is someone else's infrastructure. Your own managed server needs maintenance, monitoring, and money — forever. If you stop paying, the app dies. If the company behind your infrastructure changes direction, the app dies. If you burn out and stop maintaining it, the app dies and it takes every user's data with it.

The developer pays. The user pays. And the software still doesn't last.

This is the trap: **we have traded durability for convenience, and called it progress.**

The App Store charges $99/year and 30% commission. The developer passes the cost to users through subscriptions. The niche app that would have been a one-time purchase, built once by someone who cared, becomes economically impossible. It doesn't get built. Or it gets built, briefly exists, and disappears.

The web promised to fix this. PWAs are almost the answer — but a PWA hosted at a URL dies when the hosting dies. A URL is a dependency. A domain expires. A server goes offline. The web platform itself is durable; the hosting model built on top of it is not.

There is no platform today where a developer can build something, ship it once, have it work forever on any device, charge a fair one-time price, and walk away without ongoing liability.

That platform should exist. ForeverApps is the attempt to build it.

---

## The Core Insight

An executable file — a `.exe`, a `.app`, a Linux binary — exists independent of any server. The developer ships it once. The user runs it forever, or until their hardware stops working. There is no ongoing cost to the developer. There is no server to maintain. The software is an artifact, not a service.

The web platform has extraordinary durability. Websites from the 1990s still render in modern browsers. Browser vendors treat breaking changes as existential threats. No other runtime — iOS, Android, Windows, macOS — comes close to this backwards compatibility guarantee.

But web apps require hosting. That is the only reason they are not as durable as executables.

**Remove the hosting requirement, and the web platform becomes the most durable software runtime that has ever existed.**

A `.feapp` file is the attempt to do exactly that: a self-contained, portable artifact that packages a web application with everything it needs to run — permanently, on any device, without a server, without a domain, without ongoing cost to the developer.

The analogy is ROMs. A ROM from 1994 runs today because it is a complete artifact. It doesn't phone home. It doesn't require a subscription. It doesn't expire. When the original hardware became obsolete, emulators were built to run the ROM on new hardware — because the ROM's format declared exactly what it needed. The ROM outlasted the hardware it was built for.

A `.feapp` file is a ROM for the web platform.

---

## Why Not PWA

PWAs are a step in the right direction. Installable, offline-capable, cross-platform. But a PWA is ultimately a shortcut to a URL. It still requires hosting. The developer still owns a server dependency. If the domain expires, the PWA is gone. The user's installed icon becomes a broken link.

PWAs also have a Safari problem — IndexedDB data can be evicted for sites not visited regularly on iOS. The data is not truly the user's.

PWAs are the idea that a web app can feel like an app. `.feapp` is the idea that a web app can *be* an artifact — owned, permanent, independent.

---

## Why Not Electron

Electron is the closest existing technology to this vision, and the Electron team's reasoning for betting on the web is correct. But Electron has one architectural flaw that makes it wrong for this purpose: it bundles a full copy of Chromium with every application.

The result: every Electron app is 100MB+, every app ships its own browser, and every app is independently distributed. The architecture is `OS → [app + Chrome] × N`.

ForeverApps inverts this. One runner. One webview. Many `.feapp` files. The architecture is `OS → [ForeverApps runner] → [many .feapp apps]`.

This is how executables relate to an operating system. The OS provides the runtime. The application is just code. ForeverApps provides the runtime. The `.feapp` file is just code.

Electron also cannot run on mobile — an Electron app cannot be opened on a phone without a native port. A `.feapp` file runs on mobile through the cloud library, which serves the same file through a browser interface. No recompilation. No separate build. The same artifact.

---

## Why Not the App Store

The App Store is a distribution monopoly with a toll. $99/year to be allowed to distribute software to users who own the device in their pocket. 30% commission on every transaction. Mandatory approval process. Rules that change. Software that stops working when the developer stops paying the annual fee.

This is the mechanism that makes niche apps economically impossible. A developer who wants to build something useful for a small community, charge a fair one-time price, and move on — cannot do this through the App Store. The ongoing costs require ongoing revenue. Ongoing revenue requires subscriptions. Subscriptions require a large enough user base to sustain them.

ForeverApps does not require permission from a platform to distribute. A `.feapp` file can be shared, sold, hosted, archived, copied, and passed between people like any other file. Distribution is not a gatekeeping problem.

---

## The Steam Analogy

Steam did not invent the Windows executable. The `.exe` existed and worked before Steam. Steam organized discovery, purchase, and updates on top of an artifact format that was already complete.

This is the exact sequence ForeverApps follows:

1. The `.feapp` format is the executable. Self-contained, portable, complete.
2. The runner is the operating system layer that opens `.feapp` files.
3. The library is Steam — discovery, management, updates — built on top of a format that works without it.

The format does not depend on the library. A `.feapp` file works without ForeverApps existing at all — the runner is open source, the format is an open spec, and anyone can build a runner. This is the durability guarantee: piratability, copyability, and openness are not risks to manage. They are the mechanism of survival.

---

## What "Forever" Means

**Your data is forever yours.** All data is stored on the user's device. Always exportable to open formats. No account required to access what you created. If ForeverApps disappeared tomorrow, your data would still be usable.

**Your purchase is forever yours.** No subscription expiration. No server shutdown that kills the app. No forced upgrades. The version you bought works until the hardware it runs on stops working.

**The web platform is the most durable runtime that exists.** Browser vendors treat breaking changes as existential threats — every breaking change breaks a piece of the actual internet, so they almost never ship them. Building on the web platform is the strongest longevity bet available in software. The manifest format records exactly what runtime version an app was built for, so a future runner can emulate the right environment even decades from now.

"Forever" does not mean "never changes." Apps ship new major versions. Users choose whether to upgrade. The model is Sublime Text: pay once, use forever, pay again for the next version if you decide it's worth it.

---

## What Came Before

ForeverApps did not invent these ideas. It assembles them:

- **Unhosted / remoteStorage** (2011) — got the architecture right, never built the apps
- **Local-First Software — Ink & Switch** (2019) — articulated the philosophy precisely
- **Web Bundles — Google/W3C** (2019) — got the portable format right, stalled for political reasons not technical ones
- **Firefox OS** (2013) — got the packaged web app concept right, died with the platform
- **Electron / Tauri** — got the native shell for web apps right, without opinions about data ownership
- **PWA movement** — got cross-platform distribution right, didn't solve hosting dependency

The specific combination — shared runner, self-contained artifact, remoteStorage as the sole communication contract, manifest as time capsule, piratability as durability — does not exist. That combination is what ForeverApps is.

---

## Who This Is For

**Developers** who want to build something that lasts without ongoing infrastructure liability. Who want to charge a fair price without App Store commissions. Who want to ship once and be done.

**Users** who are tired of subscriptions, tired of their data living on someone else's server, tired of software that stops working when the company behind it changes direction.

**Hardware designers** who ship a physical product and don't want to maintain iOS, Android, and desktop apps separately — or worry about their companion software breaking on the next OS update.

**The niche app that would never get built otherwise** — because the economics of subscriptions don't work at small scale, and the economics of one-time purchases do.

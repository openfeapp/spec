# The Model

This document explains what `.feapp` fundamentally is — not the file format, not the API, but the underlying model of computation it is built on. Understanding the model makes every design decision in the spec feel obvious rather than arbitrary.

---

## Start with what you already know

You have built a web app, or you are thinking about building one. At some point you hit a wall that felt familiar. You needed the app to do something in the background — check for new data, send a notification, sync something — even when the user wasn't looking at it. And you realized: there's nowhere to put that. Your app is just a webpage. It stops existing the moment the user closes the tab.

So you reached for a server. Or a cron service. Or a job queue. Or a managed backend platform. You added infrastructure to hold the thing your app needed to be — something that keeps running.

Then you needed real-time updates pushed to the UI when something changed. Another service. Then you needed the user's data stored somewhere. A database. Then you needed that database to sync to the user's other devices. More logic. More infrastructure. More things to pay for, maintain, and watch break.

This is not a tooling problem. You didn't pick the wrong framework. This is a model problem. The model you were building in — the request/response model — was never designed to hold continuous, stateful, user-owned computation. It was designed to serve web pages to millions of anonymous users. You were trying to do something fundamentally different with the wrong model.

---

## The model you were reaching for

Here is what you actually wanted, stated plainly:

> A thing that belongs to the user. That keeps running. That owns the user's data. That does work on a schedule. That talks to the UI when something changes.

That's it. That's the whole intuition. You wanted something that behaves like a small, personal server — but one that belongs to the user, not to you, and that you don't have to maintain.

This intuition is not naive. It is not a simplification. It is a precisely correct description of a model of computation that has existed for fifty years, that powers some of the most reliable software ever built, and that has a name.

It is called the **actor model**.

---

## What an actor is

An actor is simple. It is a thing that:

- runs continuously
- has its own private memory
- receives messages and responds to them
- can send messages to other things

That's the whole model. No shared state. No request that arrives and is forgotten. Just a running thing with memory that communicates by message.

You did not need to read a computer science paper to arrive at this. You arrived at it by thinking about what your app needed. Developers have been arriving at this same intuition independently for decades. In 1973, Carl Hewitt formalized it into a theory. In the 1980s, Ericsson built a programming language called Erlang around it to run telephone switching systems. Those systems ran unattended for years. Some of them are still running. The actor model is not a trend. It is a proven, stable way to structure computation that needs to be reliable over time.

---

## The four parts of a `.feapp` app

A `.feapp` app has four parts. Each one maps directly to a concept in the model.

```
Stateful worker     the actor
                    runs continuously, independently of the frontend
                    owns the app's data lifecycle
                    processes messages from the frontend
                    does scheduled work on the user's behalf

feapp.storage       the actor's memory, externalized
                    durable, synced across devices
                    the source of truth for everything that must persist
                    owned by the user, not by any server

Stateless worker    pure functions
                    on-demand computation requested by the frontend
                    no persistent state, no memory between calls
                    fast, concurrent, always available

Frontend            the UI that talks to the actor
                    what the user sees and touches
                    sends messages to the stateful worker via IPC
                    invokes stateless functions for on-demand computation
                    reads and watches feapp.storage directly for live data
```

The frontend is the hub. Workers are spokes. The stateful worker runs whether or not the frontend is open — it is not a servant of the UI. The UI connects to it when the user opens the app, and disconnects when they close it. The worker continues regardless.

---

## How this differs from what you are used to

Most web frameworks and deployment platforms are built on the **request/response model**. A user's browser sends a request. A server handles it. A response goes back. The server forgets everything. State is stored externally — in a database the developer rents, on infrastructure the developer maintains.

This model is excellent for what it was designed for: serving millions of anonymous users from shared infrastructure. It is a poor fit for personal software — apps that belong to one user, run on their behalf, and must work even when the developer has moved on.

Here is the concrete difference in practice:

| What your app needs | Request/response approach | .feapp approach |
| --- | --- | --- |
| Run a task every 30 minutes | External cron service | `feapp.schedule.every()` |
| Push an update to the UI | External WebSocket service | `feapp.ipc.broadcast()` |
| Store the user's data | Rented database + auth layer | `feapp.storage` |
| Work across the user's devices | Sync library + conflict logic | Built into `feapp.storage` |
| Work offline | Service worker + complex setup | Built in |
| Keep working after you stop | Impossible — server must run | Yes — the file runs forever |

In the request/response world, each row in that table is a separate infrastructure decision — a vendor, a cost, a thing to maintain. In the `.feapp` world, each row is a built-in capability of the model.

This is not because `.feapp` hides the complexity. It is because the actor model is the right shape for this problem. When the model fits the problem, the complexity disappears — not into a hidden layer, but genuinely away.

---

## What the stateless worker is for

Not every computation needs to be stateful. Many things the frontend needs are pure: summarize this text, parse this file, estimate this value, transform this data. Running these inside the stateful worker would block it from processing other messages. Running them in the frontend would expose API keys.

The stateless worker handles this. It is a set of exported functions the frontend can call by name. Each call is independent. Multiple calls can run concurrently. The stateless worker has no memory between calls — it receives input, computes output, returns. This is the right model for on-demand computation.

The stateless worker reads `feapp.storage` but cannot write to it. Writes belong to the stateful worker, which is the single owner of the app's data. This is not an arbitrary restriction — it is what prevents two concurrent stateless invocations from corrupting shared state.

---

## What this makes possible

Because the stateful worker is an actor — always running, owning its data, communicating by message — a class of applications becomes possible that is difficult or impossible to build with request/response infrastructure:

**Apps that outlive their developers.** The stateful worker has no external dependencies. It talks to `feapp.storage`, which is backed by the remoteStorage protocol — a simple, open HTTP standard anyone can implement. An app built today can run in thirty years on a runner built then, reading the same manifest, providing the same environment. No server to maintain. No hosting bill. No vendor to go out of business.

**Apps that work for the user when the user isn't there.** A feed reader that syncs in the background. A finance app that reconciles accounts overnight. A monitoring tool that checks something every five minutes and alerts the user when it changes. These are natural in the actor model. They require significant infrastructure in the request/response model.

**Apps that feel instant.** Because the stateful worker is always running and connected, pushing data to the frontend is as simple as `feapp.ipc.broadcast()`. There is no polling. There is no latency from a cold start. The actor is already there.

**Apps that the user truly owns.** The user's data lives in `feapp.storage`, which can be backed by their own remoteStorage server, or by any conformant provider they choose. The app developer has no access to the data. The data does not disappear if the developer's company shuts down. This is ownership in the literal sense.

---

## What `.feapp` is, precisely

A `.feapp` file is a **local-first actor system packaged as a portable artifact**.

*Local-first* — the user's device is the primary location for compute and storage. Sync is a feature. Network connectivity is optional. The app works fully without the developer's infrastructure.

*Actor system* — the stateful worker is an actor. One per user. It owns its state, processes messages, runs continuously. The frontend and stateless worker communicate with it by message. There is no shared memory, no global state, no request that is forgotten.

*Portable artifact* — the entire system is packaged into a single file. The manifest declares exactly what environment is needed. A runner reads the manifest and provides that environment. The file runs on any conformant runner, today and in the future.

This combination — local-first, actor model, portable artifact — does not exist elsewhere. Individual pieces exist: some platforms implement actor-like models in the cloud, some protocols define local-first sync, some formats define portable apps. The specific combination, with the user as the owner and the file as the product, is what `.feapp` is.

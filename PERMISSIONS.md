# Permission Model

## Philosophy

Every `.feapp` file declares exactly what it needs. The runner enforces exactly what was declared. Nothing runs that wasn't declared. Nothing is blocked that was declared and approved.

This is not a best-practice convention. It is structural enforcement. The runner does not trust the app to stay within bounds — it enforces the bounds at the system level.

The permission model serves two purposes:

1. **User trust** — users see exactly what an app will do before it does anything. No surprises.
2. **Developer honesty** — developers are forced to declare their app's requirements explicitly. Scope creep in the manifest is visible to users.

---

## Permission Presentation

Before a `.feapp` runs for the first time, the runner presents a permission summary:

```
My Feed Reader wants to:

NETWORK ACCESS
  ✓ Contact feeds.example.com
  ✓ Contact *.rss.com
  ✗ No other network access

BACKGROUND WORKER
  ✓ Run a Deno worker process
  ✓ Worker can contact feeds.example.com
  ✗ Worker cannot access your filesystem

STORAGE
  ✓ Store data locally in IndexedDB
  ✓ Export your data as JSON anytime
  ✓ Sync via remoteStorage (optional, you control this)

OPTIONAL SERVICES
  ? Ollama at localhost:11434 (for article summarization — not required)

[Allow and Open]  [Cancel]
```

The user approves once. Permissions do not re-prompt unless the app is updated and the new version declares additional permissions.

---

## Frontend Permissions

### Network access

The runner enforces a Content Security Policy at the webview level based on the manifest declaration. The frontend cannot contact any domain not listed in `frontend.network.allowed`.

```json
"frontend": {
  "network": {
    "allowed": ["api.openweathermap.org"],
    "localhost": [11434]
  }
}
```

An empty `allowed` array means the app is fully offline. This is the default for games and simple productivity tools.

`localhost` ports are declared separately and explicitly. The frontend cannot contact arbitrary localhost ports — only those declared.

### What the frontend always has

- Access to IndexedDB for local storage
- Access to the File System Access API (user-initiated file operations only — the user must explicitly open/save)
- Web platform APIs that do not require network: Web Audio, Canvas, WebGL, WebGPU, Gamepad API, Web MIDI, Web Serial, Web USB, Web Bluetooth

These are not declared in the manifest because they require explicit user gestures (the browser's own permission model handles them). The runner does not restrict them further.

### What the frontend never has

- Arbitrary network access beyond what is declared
- Access to the local filesystem beyond user-initiated file picker operations
- Access to localhost ports not declared in the manifest
- Direct communication with the background worker outside the runner-managed channel

---

## Worker Permissions

The background worker is a Deno process. Deno's permission system is used directly, configured by the runner based on the manifest declaration.

### Network

```json
"worker": {
  "permissions": {
    "network": {
      "allowed": ["api.example.com", "*.rss.com"]
    }
  }
}
```

The worker cannot contact any domain not listed. Deno's `--allow-net` flag is set to the exact list of declared domains.

### Filesystem

```json
"worker": {
  "permissions": {
    "filesystem": {
      "read": ["~/Documents/myapp"],
      "write": ["~/Documents/myapp"]
    }
  }
}
```

The worker cannot read or write any path not declared. Paths must be specific — wildcards covering entire drives or system directories are rejected by the runner.

### System

Workers do not have access to:
- Environment variables (unless explicitly declared and approved)
- Running subprocesses
- System information beyond what is necessary for declared functionality

### Docker workers

When a worker declares `"runtime": "docker"`, the permission model changes. The runner cannot enforce Deno-level permissions on a Docker container — the container is responsible for its own sandboxing.

Docker workers are shown a different permission prompt:

```
This app uses a Docker container as its background worker.
ForeverApps cannot enforce network or filesystem restrictions
on Docker containers. Only run Docker workers from developers
you trust.

The container will be given access to:
  - remoteStorage communication channel
  - Declared network access (enforced by container configuration)
```

Docker workers are a power-user feature. The runner makes the reduced enforcement explicit.

---

## Optional Dependencies

Optional dependencies are services the app can use but does not require. The runner checks for availability on launch.

```json
"optional_dependencies": {
  "ollama": {
    "endpoint": "http://localhost:11434",
    "required": false,
    "purpose": "Article summarization",
    "models": ["llama3.2"]
  }
}
```

The runner checks if Ollama is running at the declared endpoint. If yes, it informs the user that the feature is available. If no, it informs the user that the feature will be unavailable but the app will still work.

Optional dependencies are always shown in the permission summary, even when unavailable. The user knows upfront that the app is capable of connecting to Ollama, whether or not it is currently running.

---

## Permission Changes Between Versions

When a `.feapp` is updated to a new version, the runner compares the new manifest's permissions against the previously approved permissions.

If permissions have changed:
- New permissions are highlighted and require explicit re-approval
- Removed permissions are noted (the app is requesting less access than before)
- Unchanged permissions do not require re-approval

The user is never silently granted additional permissions through an update.

---

## The Runner Cannot Be Bypassed

Permissions are not implemented inside the app. They are enforced by the runner at the system level:

- Network restrictions are implemented as webview-level CSP rules
- Worker network restrictions are implemented as Deno `--allow-net` flags at process start
- Worker filesystem restrictions are implemented as Deno `--allow-read` / `--allow-write` flags at process start
- localhost port restrictions are implemented in the runner's webview proxy

An app cannot grant itself additional permissions at runtime. An app cannot modify its own manifest. If an app attempts to contact a domain not in its manifest, the request is blocked — not logged and allowed, blocked.

# Frontend Platform Floor — feapp

This directory contains the normative frontend platform floor for feapp spec version.

The floor defines the minimum web platform surface that every conformant runner must
guarantee in its webview environment. These files are `compat-requirements/v1` artifacts
consumable by `@openfeapp/web-compat` tooling.

## Files

```
floor.language.requirements.json    ES2022 language built-ins
floor.api.requirements.json         Web platform APIs
floor.html.requirements.json        HTML elements
floor.css.requirements.json         CSS features
```

## How the floor is used

The feapp CLI consumes these files automatically at packaging time. The developer does
not pass them manually. The workflow:

1. `@openfeapp/web-compat-findings` scans the app's built frontend assets and emits a
   `compat-findings/v1` artifact (`compat.findings.json`).
2. `compat-generate-lock` from `@openfeapp/web-compat` combines the findings with the
   spec's floor requirements to produce a `compat-lock/v1` artifact (`compat.lock.json`).
3. The lockfile is embedded in the `.feapp` archive and referenced by `frontend.compat_lock`
   in the manifest.

App requirements already satisfied by the floor are omitted from the lockfile's top-level
`requirements` list. The lockfile only captures the delta above the floor.

## Derived browser floors

The floor is defined by API requirements, not browser version numbers. The minimum browser
versions implied by this floor are derived by resolving these requirements against BCD.
As of the BCD snapshot used at publication:

```
Chrome ≥ 98     (structuredClone — Feb 2022)
Firefox ≥ 111   (FileSystemFileHandle — Mar 2023)
Safari ≥ 16     (public class fields, full implementation — Sep 2022)
```

These version numbers are informational. The normative definition is the requirement files.

## Evolution

The floor is tied to the spec version. A new spec version may raise the floor but must not
remove any existing requirements — the floor is monotonic across spec versions. The floor for
version N is always a subset of the floor for version N+1. The floor for a given spec version
never changes after publication.

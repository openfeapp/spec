# Signing & Trust Model

## The Problem

`.feapp` files are designed to be copied, shared, archived, and distributed freely. That is a feature, not a risk. But it creates a question: how does a user know that a `.feapp` file actually came from the developer it claims to be from, and that it hasn't been tampered with?

The answer is signing. But signing requires trust anchors — some authority that says "this key belongs to this developer." That trust anchor is, by necessity, centralized at some level.

ForeverApps handles this the same way the web handles HTTPS: **centralized trust anchors, with the ability to distrust compromised authorities and specify alternative ones.** The system is not trustless. It is transparent, auditable, and not controlled by a single point of failure.

---

## What Signing Guarantees

A valid signature on a `.feapp` file means two things:

1. **Authorship** — this file was produced by the developer whose key is registered with the signing authority
2. **Integrity** — this file has not been modified since it was signed

A valid signature does **not** mean the app is safe, useful, or behaves as advertised. Signing verifies origin, not trustworthiness. Users should understand this distinction. The runner communicates it clearly.

---

## How It Works

### Developer key registration

A developer generates a keypair. The public key is registered with a signing authority — ForeverApps operates the default authority. The signing authority verifies the developer's identity (email at minimum) and records the association between the public key and the developer identity.

### Signing a `.feapp`

When a developer packages a `.feapp` using the CLI, the tool signs the package:

```
feapp pack ./myapp --sign --key ./myapp.private.key
```

The signature covers the entire contents of the `.feapp` package plus the manifest. Any modification to any file after signing invalidates the signature.

The signature and the developer's public key fingerprint are embedded in the manifest:

```json
"author": {
  "name": "Developer Name",
  "signing_key": "sha256:abc123...",
  "signature": "base64:xyz789..."
}
```

### Verification by the runner

When the runner opens a `.feapp` file:

1. Reads the signing key fingerprint from the manifest
2. Queries the configured signing authority for the public key associated with that fingerprint
3. Verifies the signature against the package contents
4. If valid: proceeds, displays the verified author name to the user
5. If invalid or unverifiable: warns the user clearly, requires explicit confirmation to proceed

---

## Trust Anchors

### Default signing authority

ForeverApps operates the default signing authority. The runner ships with ForeverApps as the trusted authority out of the box.

### Alternative signing authorities

The runner can be configured to trust additional signing authorities. This is relevant for:

- Organizations running a private ForeverApps deployment
- Community signing authorities if ForeverApps ceases to operate
- Self-signed apps for personal or enterprise use

```json
// runner configuration
{
  "signing_authorities": [
    "https://sign.openfeapp.org",
    "https://sign.mycommunity.org"
  ]
}
```

### If ForeverApps disappears

The signing authority endpoint is a simple, open protocol. If ForeverApps ceases to operate, a community authority can implement the same protocol and become the new default. Runners can be updated to point to the community authority. Existing signed `.feapp` files remain valid — the signature is embedded in the file, not hosted on a server.

This is the same resilience model as certificate authorities on the web. The web did not stop working when DigiNotar was compromised — it was distrusted, and traffic moved to other authorities.

---

## Key Revocation

If a developer's signing key is compromised, the signing authority publishes a revocation. The runner checks the revocation list when verifying signatures.

Revocation means:
- Future `.feapp` files signed with the compromised key are rejected
- Previously installed `.feapp` files signed with the compromised key display a warning
- The user decides whether to continue using previously installed apps

Revocation does not silently delete or disable installed apps. The user is informed and makes the decision. Their data is never affected by a key revocation.

---

## Unsigned Apps

Unsigned `.feapp` files are permitted. The runner opens them with a clear warning:

> This app is unsigned. ForeverApps cannot verify who made it or whether it has been modified. Only run unsigned apps from sources you trust.

The user explicitly confirms. There is no hard block. Enforcing a hard block on unsigned apps would recreate the App Store dynamic — a gatekeeper deciding what users are allowed to run on their own devices. That is against the core philosophy.

Unsigned apps are appropriate for:
- Personal apps built for yourself
- Apps shared within a trusted community
- Apps from developers who have not registered with a signing authority

---

## What Signing Does Not Do

- **It does not guarantee app safety.** A signed app can still behave badly within its declared permissions. Signing verifies origin, not behavior.
- **It does not prevent distribution.** A signed app can be freely shared. The signature travels with the file.
- **It does not enable remote kill switches.** ForeverApps cannot disable a `.feapp` file once it is on a user's device. Revocation affects future verification, not existing installations.
- **It does not require an internet connection to run.** The runner caches verification results. An app that has been verified runs offline without re-verification.

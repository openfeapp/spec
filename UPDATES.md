# Update Protocol

## Philosophy

A `.feapp` file is an immutable artifact. It does not update itself. The version you have works forever — that is the guarantee.

Updates are separate artifacts. A new version of an app is a new `.feapp` file, produced by the developer, signed, and made available through an update feed. The library discovers new versions, notifies the user, and downloads the new file. The user decides when to switch. The old version is kept and continues to work.

The update mechanism is a convenience layer on top of an artifact format that does not require it. If the update feed disappears, existing `.feapp` files are unaffected.

---

## Update Feed Format

The `distribution.update_feed` field in the manifest points to a JSON file hosted by the developer. The library polls this URL to check for new versions.

```json
{
  "feapp_update_feed": "0.1",
  "app_id": "com.developer.myapp",
  "releases": [
    {
      "version": "2.0.0",
      "released": "2025-06-01",
      "download_url": "https://developer.com/myapp/myapp-2.0.0.feapp",
      "signature": "sha256:abc123...",
      "size_bytes": 4200000,
      "changelog": "Major update: new templates, improved performance",
      "breaking_changes": false,
      "schema_migration": false,
      "min_previous_version": null
    },
    {
      "version": "1.1.0",
      "released": "2025-03-15",
      "download_url": "https://developer.com/myapp/myapp-1.1.0.feapp",
      "signature": "sha256:def456...",
      "size_bytes": 3800000,
      "changelog": "Bug fixes, new export options",
      "breaking_changes": false,
      "schema_migration": false,
      "min_previous_version": null
    }
  ]
}
```

### Feed fields

**`feapp_update_feed`** — the spec version of the update feed format.

**`app_id`** — must match the `id` field in the installed app's manifest. The library rejects feeds where this doesn't match.

**`releases`** — ordered array of available versions, newest first.

**`version`** — semantic version of this release.

**`released`** — ISO 8601 date.

**`download_url`** — direct URL to the `.feapp` file for this version.

**`signature`** — SHA-256 hash of the `.feapp` file at `download_url`. The library verifies this before presenting the download to the user.

**`size_bytes`** — size of the download. Shown to the user before they confirm.

**`changelog`** — human-readable description of what changed. Shown in the library's update notification.

**`breaking_changes`** — boolean. If true, the library highlights that this update may change how the app behaves.

**`schema_migration`** — boolean. If true, the app's data schema is changing in this version. The library warns the user and prompts them to export their data before upgrading.

**`min_previous_version`** — if set, the library will only offer this update to users running this version or later. Used for updates that require a migration path.

---

## Library Update Behavior

### Polling

The library polls update feeds on a schedule configured by the user (default: once per day). Polling is done in the background. The user is not interrupted.

### Notification

When a new version is available, the library displays a notification in the app's library entry:

```
My Journal  v1.0.0  →  v1.1.0 available
Bug fixes, new export options
[View details]  [Update]  [Skip]
```

The user is never automatically updated. Updates require explicit user action.

### Download and verification

When the user confirms an update:

1. The library downloads the new `.feapp` file from `download_url`
2. Verifies the file hash against the `signature` field in the feed
3. Verifies the developer signature against the signing authority
4. If both pass: proceeds
5. If either fails: rejects the download and notifies the user

### Version coexistence

After a successful update download, the library keeps the previous version. Both versions are available in the library. The user switches to the new version explicitly.

The old version is never deleted automatically. Users can delete old versions manually when they are confident the new version works for them.

### Schema migrations

If `schema_migration` is true in the update feed:

1. The library warns the user: "This update changes how your data is stored. Your data will be migrated automatically. It is recommended to export your data before proceeding."
2. The user can export their data as JSON before confirming
3. On first launch of the new version, the app runs its migration routine against the local data
4. If migration fails, the library rolls back to the previous version and the user's data is unaffected

---

## Self-Hosted Update Feeds

The update feed is a static JSON file. It can be hosted anywhere:

- A personal website
- A GitHub repository (raw file URL)
- An S3 bucket or equivalent
- The ForeverApps directory (if listed)

The developer owns the update feed. ForeverApps does not control it, cannot modify it, and cannot prevent updates from being delivered.

---

## Updates Without an Update Feed

Apps without a `distribution.update_feed` field do not receive automatic update notifications. Users can still manually replace a `.feapp` file in the library by dragging in a new version. The library will detect the version difference and present the same update confirmation flow.

This covers apps distributed informally — shared directly between people, without a hosting infrastructure.

---

## Security Considerations

The update mechanism is a potential attack surface. The library defends against:

**Downgrade attacks** — the library will not present a version older than what is currently installed as an "update." Older versions can only be installed manually.

**Feed tampering** — the `signature` field in the feed is verified against the downloaded file before the user sees the update. A compromised CDN or download URL cannot deliver a modified file without the signature failing.

**Feed hijacking** — the feed URL is declared in the original manifest and signed with the developer's key. Changing the feed URL requires a new signed `.feapp` release. The library does not follow redirects to different domains without user confirmation.

**Signing authority compromise** — if a developer's signing key is revoked, update downloads signed with that key are rejected. See `SIGNING.md`.

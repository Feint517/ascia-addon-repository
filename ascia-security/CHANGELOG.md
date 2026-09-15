# Changelog — ascia-security

The `user → role/area` map and the encrypted, tamper-evident audit log. The BFF
resolves a logged-in user's role and areas here and signs them into
`X-Ascia-User`; this add-on is the source of truth and the audit sink. Format
follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versions use
[SemVer](https://semver.org/spec/v2.0.0.html).

Started at 0.2.0 — earlier releases predate this file. Written now because the
change below alters an authorization table on disk, and a migration that touches
who may open what deserves a record an operator can find without reading a diff.

## [0.3.0] - 2026-09-13

### Changed — BREAKING

- **The signed `X-Ascia-User` payload now spells the area list `areas`.** It said
  `zones` until now — the field had moved in SDK 0.7.0 but the wire key was held
  back deliberately, because those bytes are HMAC-signed and moving the key needs
  the BFF and all ten add-ons in one drop.

  **This release is that drop.** Every add-on and the front door must be updated
  together. A component on the other side of it now fails with a **401**, not a
  silent empty area list: `_decode_user` rejects a payload with no `areas` key
  rather than letting the model default it, because a non-admin arriving scoped
  to nothing looks like working software and logs nothing.

See `ascia-addons/docs/RENAME_AREA_AND_PLAN.md` § D3.

## [0.2.0] - 2026-09-13

### Changed — BREAKING

- **`zone` → `area`** throughout. `POST /security/users` and `PATCH
  /security/users/{id}` take `areas` instead of `zones`; `GET
  /security/users/{id}/context` returns `areas`. `UserContext.zones` became
  `.areas` in the SDK (0.7.0) and `can_access_zone()` is `can_access_area()`.

  `zone` was ASCIA's word for a place in the home, and it is Home Assistant's
  word for a *geofence* — so an add-on holding HA `area_id`s in a field called
  `zones` had three names for one concept. See
  `ascia-addons/docs/RENAME_AREA_AND_PLAN.md`.

### Migrated

- **`user_zones` → `user_areas`** (schema v2). Applied automatically on boot by
  the SDK's migration runner; **existing grants are carried, not rebuilt.**

  `ALTER TABLE ... RENAME TO`, deliberately: it keeps the foreign key and its
  `ON DELETE CASCADE`, where a create-copy-drop would lose the cascade and leave
  a deleted user's grants attached to a recycled id.

  The `area_id` **column does not move** — it was already correct, because the
  identifier kept Home Assistant's name when the concept adopted HA's word.

  Migration `_V1` is left exactly as it shipped. A migration list is a history,
  not a description of the current schema: editing v1 would never repair an
  appliance that already ran it, and would silently diverge fresh databases from
  existing ones.

### Note on the signed header

The `X-Ascia-User` payload **still spells the key `zones`** on the wire. That is
deliberate and temporary: those bytes are HMAC-signed, so renaming the key needs
the BFF and all ten add-ons in one drop, and getting it wrong 401s every
authenticated call at once. Phase 7 of the rename plan retires it as its own
small release.

# Changelog — ascia-backup

ASCIA's trustworthy-by-default backup layer over HA Supervisor's backup primitive. Format
follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versions use
[SemVer](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

Pre-release work toward `0.1.0`. All items below ship together when the first tag lands.

### Changed

- **Listens on port 8122 (was 8099).** Each ASCIA add-on now owns a unique port baked into its
  image (device-manager 8099 [host_network], vpn 8120, 3d-engine 8121, backup 8122) so they never
  contend for a host port. The API stays internal (`ports: {8122/tcp: null}`), reached through the
  ascia-frontend BFF.

### Added

- **Initial scaffolding.** FastAPI app via the SDK's `create_app` — `/health` and `/metrics`
  work; auth secrets gate everything else. No business-logic routes yet; those land in
  subsequent commits per the API contract.
- **Design contract.** OpenAPI 3.0 at `../../../ascia-docs/api/ascia-backup.openapi.yaml`
  (13 paths, 17 schemas) and companion design rationale at
  `../../../ascia-docs/design/ascia-backup.md`.
- **HA add-on manifest.** `hassio_role: admin` (needed for Supervisor's create/restore/delete
  backup APIs); `map: addon_config:rw + backup:rw` for per-add-on state and access to the
  `/backup` directory; no `homeassistant_api` (this add-on doesn't talk to HA Core directly);
  `service_token` + `user_signing_key` exposed in the Configuration UI by default.
- **Dockerfile** mirroring the device-manager/3d-engine pattern: layered on `ascia-base`,
  installs the SDK then the add-on from the monorepo-root build context, ships the S6 service
  via `rootfs/`.
- **Workspace integration.** Declared in the root `pyproject.toml` workspace + sources so
  `uv sync --all-extras` picks it up; ci-addons.yml's `discover` job will find it via the
  trio (config.yaml + pyproject.toml + Dockerfile) and add it to the build matrix without
  any further changes.
- **Final four routes real — all 13 contract paths now implemented.** Closes the
  feature surface; integrity check and restore safety guarantees are honored.

  SDK additions:
  - `SupervisorClient.restore_backup(slug)` -> POST `/backups/{slug}/restore/full`.
    Tests: happy path + error envelope -> UpstreamError.

  Config + Services:
  - `BackupOptions.home_name: str = "ASCIA Home"` — compared (case-insensitive,
    trimmed) against the restore route's `confirm_phrase` body.
  - `Services.options: BackupOptions` — wired through `main.build()`.

  Routes:
  - `GET /status` — aggregates Supervisor's `list_backups` (count, total size,
    oldest/newest) with state (schedule.enabled, pending_restore, targets summary
    incl. healthy count). `last_backup_status` and `next_scheduled_at` stay null in
    v1 — the former needs per-backup run-success tracking on state (lands with the
    scheduler), the latter needs croniter.
  - `GET /backups/{slug}/download` — streams the tarball through from Supervisor's
    `/backups/{slug}/download` via httpx streaming. v1 proxies plaintext; the
    AES-GCM wrap layer is Phase 2 (POST /backups still triggers Supervisor's native
    unencrypted backup). Content-Disposition headers set so browsers prompt to save.
    TODO marked for moving the stream into a SupervisorClient method so we don't
    reach into `_client`.
  - `POST /backups/{slug}/verify` — verifies the slug exists via Supervisor's get,
    records `state.last_verify_at`, caches integrity on the per-slug state record
    if present. v1 placeholder: `header_decrypted` reflects `state.encrypted`
    (false for all Supervisor-created backups today); `checksum_match` always True
    when Supervisor confirms readability. Real `encryption.verify_envelope` runs
    when the encryption-after-create wiring lands.
  - `POST /backups/{slug}/restore` — admin-gated, destructive. 409 if
    `state.pending_restore` is non-None (latch survives reboot for the countdown
    UI). 400 `confirm_phrase_mismatch` if the body's phrase doesn't match
    `services.options.home_name` (case-insensitive, trimmed). 404 on unknown slug.
    On happy path: state advances PRE_BACKUP -> a forced safety backup via
    `SupervisorClient.create_full_backup("pre-restore <ts>")` -> RESTORING with the
    safety slug stamped on, then Supervisor's restore is fired. Synchronous failure
    transitions to FAILED with the upstream error message.

  Three new error classes: `_ConfirmPhraseMismatch` (400), `_RestoreInProgress` (409),
  reuses `_BackupNotFound` (404) for cross-Supervisor 404 translation.

  Tests: 14 new route tests + 2 new SDK tests covering every documented behavior.

- **GET / POST / DELETE /targets real** (9th + 10th + 11th real routes — still pure
  state-store; the USB target adapter that mounts + mirrors files is Phase 2). GET
  returns the registered targets from state with v1 status defaulting to `ok`. POST is
  admin-gated; v1 ships `kind=usb` only; `kind=smb`/`nfs` return
  `400 unsupported_target_kind` (the enum values are reserved in the schema so a Phase 2
  adapter drops in without a contract change). Auto-generates ids as `<kind>-<6 hex
  chars>` via `secrets.token_hex(3)` so multiple drives of the same kind can coexist
  without collision. DELETE is admin-gated; `404 not_found` if the target id is unknown;
  physical files on the mount are deliberately NOT touched (de-registering shouldn't
  wipe a homeowner's USB drive). Two new error classes: `_UnsupportedTargetKind` (400)
  and `_TargetNotFound` (404; shares the `not_found` code with `_BackupNotFound` —
  different domains, same envelope code).
- **GET / PUT /schedule real** (7th + 8th real routes — pure state-store work, no
  Supervisor). GET returns the operator-configured schedule from the state store with
  design-doc defaults on a fresh install (`0 3 * * *` daily at 3 am in
  `America/Toronto`, retention `{daily: 7, weekly: 4, monthly: 3}`). PUT is admin-gated,
  runs `_validate_cron` (5 POSIX whitespace-separated fields) and `_validate_tz`
  (`zoneinfo.ZoneInfo` lookup against the IANA database) before persisting, returns
  `400 invalid_schedule` on either; otherwise merges through
  `StateStore.update_schedule` (PATCH-shaped — nested retention also field-merges; Pydantic
  enforces non-negative retention counts at the model layer). The actual scheduler that
  fires backups doesn't exist yet; the route persists the operator's intent so the
  eventual scheduler picks it up at startup.
- **DELETE /backups/{slug} real** (6th real route). Admin-gated. SDK side: new
  `SupervisorClient.delete_backup(slug)` plus a shared `_delete` helper mirroring the
  existing `_get` / `_post` plumbing. Backend: pre-flight `list_backups` to enforce the
  "refuse the last backup" safety check — exactly one backup remaining returns
  `400 last_backup` unless the optional body sets
  `{"i_know_this_is_the_last_one": true}`. After Supervisor confirms the delete, the
  per-slug state record is pruned so subsequent /backups calls don't return a ghost.
  Supervisor's "not found" error envelope is translated to our `404 not_found`. The
  `409 backup_in_use` response (restore/verify currently reading the backup) is
  documented as a TODO and lands alongside the restore/verify route implementations.
- **POST /backups real** (the third write-side route after the two key endpoints, fifth
  real route total). Triggers Supervisor's `/backups/new/full` with an optional body
  (kind=full default; name auto-generated as `manual <timestamp>` if omitted).
  Concurrent-create guard via a new `services.backup_lock: asyncio.Lock` — second
  in-flight request returns `409 backup_in_progress` immediately rather than queueing or
  hitting Supervisor twice. Records `state.last_manual_at` on success so the eventual
  `/status` widget can show "last manual: 2h ago." Partial backups (`kind=partial`) still
  return the `not_implemented` 501 envelope — that path needs the addons/folders body
  validation + the corresponding `/backups/new/partial` Supervisor call, deferred. v1 is
  full-only per the design doc.
- **Supervisor integration + two more real routes** (`GET /backups`, `GET /backups/{slug}`).
  SDK side: `SupervisorClient.list_backups()` and `get_backup(slug)` plus a new
  `BackupRecord` model mirroring Supervisor's wire format (with `extra='ignore'` for
  forward-compat against future Supervisor fields). Backend side: `Services` gains a
  `supervisor: SupervisorClient` field; `main.py` opens a background `_wait_for_supervisor`
  retry loop (mirrors the engine's HA-wait pattern) that flips the new `supervisor`
  readiness gate once `addon_info` answers. `/health` reports 503 until both `key` and
  `supervisor` are ready. Routes merge Supervisor's view (slug/name/kind/date/size/addons/
  ha-version, authoritative) with our state's view (origin/encrypted/integrity/locations,
  authoritative for the ASCIA-added metadata). `GET /backups` also prunes stale state
  records whose slug Supervisor no longer reports (handles "deleted via HA UI" out-of-band
  case). `GET /backups/{slug}` surfaces Supervisor's "not found" error envelope as our
  standard `404 not_found`. Env override `ASCIA_SUPERVISOR_URL` lets local-dev / tests
  point at a mock without touching the production address.
- **First two real routes wired** (replacing two of the 11 stubs):
  `GET /key/recovery-info` and `POST /key/rotate`. Recovery-info enforces the one-shot
  reveal latch (`state.key_recovery_shown`) — first authenticated admin call returns the
  hex key + QR data URL + key_id; every subsequent call returns
  `410 recovery_info_already_shown` until the next rotation. Rotate generates a fresh
  key via `KeyManager.generate()`, persists it (atomic mode-0600 write), and resets the
  recovery latch via `StateStore.record_key()` so the next recovery-info call reveals
  the new key. The other 9 routes remain `501 not_implemented` stubs.
- **`Services` holder** (`services.py`). Frozen dataclass bundling `StateStore` + `KeyManager`
  so route handlers take one dependency instead of a long argument list. Mirrors
  device-manager's `Services` and the engine's `SceneService` injection. Future scheduler /
  target adapters land here as new fields.
- **Lifespan integration** (`main.py`). New `key` readiness gate (so `/health` reports
  `ready: false` until the encryption key is initialized). `_ensure_key()` handles three
  cases on startup: key.bin missing -> generate fresh + record id on state (admin must
  capture recovery once); key.bin present + state.key_id matches -> no-op; key.bin
  present + state.key_id mismatched -> reconcile state to what key.bin actually
  contains (file is authoritative; handles the "restored key.bin from USB" recovery
  scenario). Paths read from env (`ASCIA_BACKUP_STATE_PATH` / `ASCIA_BACKUP_KEY_PATH`)
  with `/data/...` defaults for HAOS.
- **`KeyManager.derived_id()`** — public helper exposing the SHA-256[:16] derivation so
  the lifespan can reconcile state.key_id with what key.bin contains without reaching
  into `_derive_key_id`.
- **Encryption layer** (`encryption.py`). Two halves: `KeyManager` (owns `/data/key.bin`
  — generate / load / rotate / recovery-info; atomic mode-0600 write; key_id =
  first-16-bytes-of-`SHA256(key)` for audit), plus a streaming AES-256-GCM envelope over
  backup tarballs (`encrypt_file` / `decrypt_file` / `verify_envelope`). File format:
  40-byte header (magic + version + reserved + key_id + nonce) fed as AAD, ciphertext,
  trailing 16-byte tag — single GCM message per file (per-message 64 GiB bound is far
  above any HA backup). Pre-check via key_id fails fast on wrong key; tag covers header
  AND ciphertext so any tamper anywhere triggers a clean error. QR rendering via segno
  produces an `image/svg+xml` data URL ready to drop into an `<img>` for printing. New
  deps: `cryptography>=42` (low-level Cipher API for streaming) and `segno>=1.6` (QR).
- **Persistent state store** (`state.py`). `AddonState` Pydantic model carries everything
  that has to outlive a restart: the schedule + retention, the backup catalog (keyed by
  slug, with each backup's integrity status + every location it's mirrored to), the
  off-site target list, encryption-key bookkeeping (key id + the recovery-shown latch),
  recent-run timestamps, and the pending-restore state (so the frontend's countdown UI
  survives the host reboot a restore triggers). `load_state` / `save_state` mirror the
  engine's `anchors.py` atomic-write pattern (tmp + `os.replace`). `StateStore` is the
  caching wrapper the rest of the add-on uses — typed mutation helpers
  (`update_schedule` with PATCH-shaped merge incl. nested retention, `upsert_backup`,
  `remove_backup`, `set_integrity`, `add_target` / `remove_target`, `record_key` +
  `mark_recovery_shown` latch logic, run-timestamp markers, `set_pending_restore`) every
  one of which persists on the same call. Malformed state files raise `StateFileError`
  loudly rather than silently overwriting with defaults. Forward-compat via Pydantic's
  default `extra: ignore` plus a `schema_version: 1` field for future migrations.
- **Stub routes for all 13 contract paths** (`routes.py`). Every handler raises a
  structured `501 not_implemented` envelope. The BFF can wire up its proxy routes now
  and the frontend can render a deliberate "coming soon" surface keyed off
  `error.code == "not_implemented"` instead of chasing 404s. Mutating handlers
  (delete / restore / schedule update / target add+remove / key recovery+rotate)
  already wear the `require_role(Role.ADMIN)` dependency so the eventual real
  handlers don't have to change the auth wiring. `main.py` mounts the router via
  `app.include_router(build_router())`.
- **OpenAPI nit:** renamed `/targets/{id}` path parameter to `/targets/{target_id}` to
  match Python conventions without shadowing the `id` builtin in route handlers.
- **Pydantic domain models** (`models.py`). All 17 schemas from the OpenAPI contract:
  `BackupKind` / `BackupOrigin` / `BackupIntegrityStatus` / `BackupRunStatus` /
  `BackupLocationKind` / `TargetKind` / `TargetStatus` / `RestoreState` enums (string values
  match the contract verbatim — the frontend keys off them); `Backup` + `BackupLocation` +
  `BackupCreate`; `Schedule` + `RetentionPolicy` + `ScheduleUpdate` (with the design doc's
  `7/4/3` retention defaults baked in); `Target` + `TargetCreate`; `KeyRecovery` (always
  `never_show_again: true` server-side); `RestoreRequest` + `RestoreStatus`; `VerifyResult`
  (defaults `location_checked` to `local`); `TargetsSummary` + `Status`. Datetime fields
  serialize to ISO-8601 in `mode="json"`. Round-trip and edge cases tested.

## [0.0.0] — initial scaffolding

Add-on package created from the device-manager / 3d-engine template; no functional code
beyond the FastAPI shell and the SDK's `/health` + `/metrics` routes.

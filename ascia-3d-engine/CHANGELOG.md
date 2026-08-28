# Changelog — ascia-3d-engine

The 3D scene service: server-side `entity↔3D` mapping, GLB model storage, live
`/scene/subscribe` WebSocket, emergency transform. Format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versions use
[SemVer](https://semver.org/spec/v2.0.0.html).

This file is consumed by the Home Assistant add-on store when the add-on is
republished, and by anyone inspecting `/data` or the source tree on a running
add-on.

## [Unreleased]

Pre-release work toward `0.1.0`. All items below ship together when the first
tag lands.

### Changed

- **Listens on port 8121 (was 8099).** Each ASCIA add-on now owns a unique port
  baked into its image (device-manager 8099 [host_network], vpn 8120, 3d-engine
  8121, backup 8122) so they never contend for a host port. The API stays
  internal (`ports: {8121/tcp: null}`), reached through the ascia-frontend BFF.

### Added — Phase 1 functional increments

- **Scaffold + minimum viable scene.** Standard add-on shape (`config.yaml`,
  `build.yaml` `FROM ascia-base`, S6 service, `uv pip install --no-deps
  --no-sources` Dockerfile pattern, fail-fast secrets in `main.py`). Pure
  `classify(EntityState) → (DeviceKind, VisualState)` covers
  light/cover/lock/door/camera/presence/leak/smoke/sensor with defensive
  attribute coercion. `SceneService.load()` warms HA areas + entity registry
  and serves `GET /scene/zones` and `GET /scene/state`. `GET /scene/model`
  returns a structured `404 no_model` until a GLB is uploaded;
  `/scene/subscribe` is a 501 placeholder until WebSocket lands.
- **Anchors config: cascade + reload + floors.** New strict-Pydantic
  `anchors.json` schema (`AnchorsConfig`, `ZoneConfig`, `AnchorEntry`) with
  unknown-key rejection. Missing file = empty config; bad JSON / schema →
  raises `anchors_config_invalid`. Anchor cascade resolves each device's
  position by priority: explicit anchor → zone centroid → world origin, and
  stamps the choice on the response as `anchor_source`
  (`explicit` | `centroid` | `origin`) so the frontend can surface unplaced
  devices. Adds `GET /scene/floors` + `POST /scene/anchors/reload`.
- **Live updates: WebSocket `/scene/subscribe`.** New `pubsub.py` with
  `ScenePubSub` fan-out hub: bounded per-client queues (default 64); on
  overflow the subscriber's `closed` event is set and the WS handler closes
  the connection (1011) so the client reconnects to a fresh snapshot — no
  silent staleness. Background follower task maps every HA `state_changed`
  event into a `SceneDevice` and broadcasts as a `device_update`. WS handler
  verifies the bearer manually (Starlette's HTTP middleware doesn't run on WS
  upgrades), subscribes to the pubsub *before* taking the initial snapshot
  (so events landing in that gap are queued), then drains the queue.
- **GLB upload + binary serve.** New `model_store.py`: streams uploads in
  64 KiB chunks, validates magic bytes (first 4 = `glTF`), max size, and
  content-type; writes via tmp + `os.replace` for atomicity (a half-uploaded
  model can never be served). Sidecar metadata (`model.glb.json`) caches the
  content-SHA256 version + size + generated-at so `GET /scene/model` doesn't
  re-hash. Same bytes = same `model_version` → perfect HTTP cache key. New
  routes: `POST /scene/model` (multipart), `GET /scene/model` (manifest with
  floors), `GET /scene/model.glb` (streamed via FastAPI `FileResponse`).
  New error codes: `model_invalid_glb` (400), `model_too_big` (413),
  `model_unsupported_content_type` (400).
- **Emergency transform: fire/smoke/leak detection + broadcast.** New
  `emergency.py` module:
  - Pure `emergency_trigger(EntityState) → EmergencyType | None`. Maps
    `binary_sensor` `device_class` to emergency types:
    `heat`/`gas`/`carbon_monoxide` → `fire`, `smoke` → `smoke`,
    `moisture` → `leak`. Only when state is exactly `"on"`.
  - Stateful `EmergencyTracker` with priority resolution (**fire > smoke >
    leak**), focus-device sort stability, and `started_at` preservation —
    timer doesn't reset across same-id observations, only on transitions.
  - `SceneService` seeds the tracker on `load()`, re-seeds on every `state()`
    call, observes incrementally inside the follower. On a real transition
    (different type/zone/focus) the follower broadcasts an `emergency`
    SceneUpdate.
- **`POST /scene/anchors`.** Replaces the whole anchors config atomically;
  used by the frontend's placement UI. Engine writes via tmp + `os.replace`,
  reloads in memory, returns the new config. New `GET /scene/anchors` for the
  UI to download the current config.

### Added — Phase 1 hardening

- **Hardening #1 — follower-health observability.** Outer wrapper around the
  state_changed follower catches any inner-loop exception, increments
  `ascia_3d_engine_follower_restarts_total`, sleeps for exponential backoff
  (1 → 30 s), and restarts. New "events" readiness gate
  (`Readiness(["ha", "events"])`) flips false during outage so `/health`
  returns 503 instead of pretending to be ready while serving stale state.
- **Hardening #2 — HA registry-drift follower.** Subscribes to
  `area_registry_updated` / `entity_registry_updated` /
  `device_registry_updated` in parallel; on any event refreshes the cached
  entity→zone mapping and broadcasts a fresh `scene_state` so connected
  clients re-snap to the new layout without polling. New metric
  `ascia_3d_engine_registry_refreshes_total{event_type}`. Restart-on-crash
  wrapper with the same shape as the state follower.
- **Hardening #3 — Admin-only gates on mutating routes.** `POST /scene/model`,
  `POST /scene/anchors`, and `POST /scene/anchors/reload` now require
  `require_role(Role.ADMIN)` via a FastAPI dependency. Bearer alone returns
  401 (`requires user context`); non-admin signed user returns 403.
- **Hardening #4 — `X-Ascia-User` signing flowing through the BFF.** Engine
  side: routes use the SDK's existing `require_role`. BFF side (in
  ascia-frontend): all `+server.ts` endpoints + the Vite WS proxy mint a
  signed admin user header on every outbound call. Canonical-JSON encoding
  matches Python's `json.dumps(..., separators=(",", ":"), sort_keys=True)`
  byte-for-byte; verified by cross-language round-trip.
- **Hardening #5 — `verify_ws_credentials` in the SDK.** New helper exported
  from `ascia_sdk.app`: checks bearer + optional signed user from a WS
  upgrade. Engine's WS handler deletes its local `_check_ws_auth` and uses
  the SDK helper instead. Available to every future add-on with a WS
  surface.
- **Hardening #6 — emergency silence endpoint.** New admin-gated
  `POST /scene/emergency/silence` flips the current emergency's `silenced`
  flag to true (or 404 `no_emergency` if none active) and broadcasts the
  updated state. Frontend uses this to drop the alarm choreography while
  keeping the banner visible as a static acknowledgement. `silenced` is
  preserved across same-id observations; an upgrade (different type or zone)
  resets it so a more-severe event re-triggers the full alarm.
- **Engine sidebar filter — `is_renderable`.** New pure helper in
  `classify.py` keeps only physical-domain entities (`light`, `switch`,
  `cover`, `lock`, `fan`, `climate`, `vacuum`, `humidifier`, `water_heater`,
  `camera`, `binary_sensor`, `alarm_control_panel`, `media_player`, plus
  `sensor.*` *with* a `device_class`). Applied in `state()` and the live
  follower so `sun.*`, `weather.*`, `automation.*`, `script.*`, helpers,
  info-only sensors never reach the wire. Cuts a typical HA's exported
  entity count roughly in half.

### Added — observability

- Prometheus metrics: `ascia_3d_engine_follower_up`,
  `ascia_3d_engine_follower_restarts_total`,
  `ascia_3d_engine_registry_refreshes_total{event_type}`,
  `ascia_3d_engine_subscribers`,
  `ascia_3d_engine_emergency_active{type}`. Scraped at the SDK-provided
  `/metrics` endpoint.

### Added — developer ergonomics

- Local-dev env-var overrides: `HA_BASE_URL`, `HA_WS_URL`, `ANCHORS_PATH`,
  `MODEL_PATH`. Lets the engine run against the local `ascia-dev` compose
  stack without HAOS — `/data/*` is HAOS-only and read-only on macOS, so the
  overrides point at a writable dev directory.
- `tooling/scripts/dev/setup-dev-secrets.sh` — generates + aligns
  `ASCIA_SERVICE_TOKEN` + `ASCIA_USER_SIGNING_KEY` across the engine env and
  the frontend's `.env` so they're guaranteed to match by construction.
- `tooling/scripts/ops/upload-glb.py` — signed-admin GLB uploader for terminal
  workflows that don't want to drag-and-drop through the frontend.

### Changed

- `EngineOptions` gains `anchors_path` (default `/data/anchors.json`),
  `model_path` (default `/data/model.glb`), `max_model_bytes` (default
  100 MB).
- `SceneDevice.anchor_source` (required field) — `explicit` | `centroid` |
  `origin`. Lets the frontend surface "unplaced" devices.
- Default `Readiness(["ha", "events"])` instead of `["ha"]`. `/health` now
  returns 503 if *either* gate drops.

### Verified

- **177 add-on tests pass** (~155 for scene logic + 22 for the hardening
  items). Lint clean.
- OpenAPI contract at
  `../../../ascia-docs/api/ascia-3d-engine.openapi.yaml` validates via
  `openapi-spec-validator` and mirrors every shipped route.
- Cross-language round-trip verified: TS-signed `X-Ascia-User` decodes in
  Python with the same key (`signUserContext` ↔ `sign_user_context`).
- Live against a real HA on macOS:
  - `/scene/state` returns ~60 entities after the renderable filter.
  - Drag-and-drop GLB upload through the frontend produces a manifest with a
    deterministic content-SHA `model_version`.
  - Click-to-place anchor placement writes to `dev-data/anchors.json` and
    re-renders within ~50 ms.
  - Tripping `binary_sensor.basement_floor_wet` flies the camera, fires the
    vignette, and starts the timer; silencing drops the alarm visuals and
    keeps the banner; clearing the sensor in HA restores the default view.

## [0.0.0] — initial scaffolding

Add-on package created from the `ascia-base` template; no functional code
yet beyond the FastAPI shell and a `/health` route inherited from the SDK.

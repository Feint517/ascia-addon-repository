# Changelog — ascia-device-manager

Device auto-discovery, onboarding, naming, and zone assignment. Format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versions use
[SemVer](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- **Device control plane (C1-C6).** Capability-aware `POST /devices/control`
  plus `GET` and batch `POST /devices/control/capabilities`. Each command is
  validated against a per-domain catalog (light, switch, cover, climate, fan,
  lock, media_player), gated by the role+zone policy (`can_control`), executed
  via the HA service call, and the new state is read back. The batch
  capabilities endpoint lets the BFF gate every entity's controls in one call.
  Validated live against the HAOS VM (real demo entities + six simulated MQTT
  device types). Capability gating is enforced server-side; it becomes
  per-user once the BFF forwards real identity (currently signs dev-admin).

## [0.1.0]

Pre-release work toward `0.1.0`. All items below ship together when the first
tag lands.

### Added

- **In-memory `DeviceCache` warmed from HA registries + `state_changed`
  events.** Serves `GET /devices` from cache so the BFF doesn't pay a
  cross-WS round-trip on every request.
- **Composition pipeline.** `Device` aggregates `(device_id, name, area_id,
  manufacturer, model, entities, last_state)` from HA's device + entity
  registries and the live state map. Surfaces only physical devices: it lists
  device-registry devices (device-less helper/YAML entities aren't promoted —
  they can't take a zone via the device registry anyway) and drops HA "service"
  devices (`entry_type == "service"`: Sun, weather, Backup, HAOS
  Core/Supervisor/Host/OS, add-ons), mirroring how HA's own UI separates them.
- **Zigbee adapter via Zigbee2MQTT.** Subscribes to `zigbee2mqtt/bridge/*`
  topics, parses joining/leaving devices into the cache, and exposes
  routes:
  - `GET /devices/zigbee` — discovered Zigbee devices.
  - `POST /devices/discover` — open the Zigbee2MQTT permit-join window.
  - `POST /devices/zigbee/{ieee}/rename` — rename a Zigbee device end-to-end
    (Zigbee2MQTT bridge command + cached state update).
- **Zones (HA areas).** `GET /devices/zones` lists zones from the cache and
  `POST /devices/zones` creates one (via the SDK's `HAClient.create_area`,
  reflected in the cache immediately). `POST /devices/{id}/zone` assigns a
  device to a zone via `HAClient.update_device_area`. Each `Device` now also
  carries its `area_id` (alongside the resolved `area` name) so the UI can
  pre-select the current zone.
- **Network discovery (Phase 2, read-only).** A pluggable `discovery/`
  subsystem surfaces devices seen on the network — separate from HA devices,
  never writing to HA — via `GET /devices/discovered` (+ `POST .../scan`):
  - **Wi-Fi (W1):** mDNS (`zeroconf`) + SSDP (stdlib UDP) browsers; dedup/merge
    by MAC → USN/host; each entry flags `in_ha` (matched against HA's managed
    MACs) and a best-effort `suggested_zone`.
  - **ONVIF cameras (W2):** WS-Discovery + optional authenticated SOAP
    enrichment (`GetDeviceInformation`/`GetCapabilities`/`GetProfiles`/
    `GetStreamUri`, hand-rolled WS-UsernameToken) capturing make/model/firmware
    and the RTSP stream URL for the NVR; global `onvif_username`/`onvif_password`.
  - **PoE / Omada (W3):** poll-based `OmadaDiscoverer` over the TP-Link Omada
    controller's **Open API** (OAuth client-credentials → `AccessToken=` header).
    Surfaces network infrastructure (switches/APs/gateway) and wired PoE devices
    as rows, and uses the controller's physical-network view to **enrich by MAC** —
    an ONVIF/mDNS device gains its switch + port + PoE flag, or Wi-Fi signal/SSID.
    Wireless clients (phones/laptops) are enrich-only, never their own rows. No
    new dependency (httpx + json). New categories `switch`/`access_point`/
    `router`; `config.yaml` gains `omada_url`/`omada_id`/`omada_client_id`/
    `omada_client_secret`/`omada_site`. Field shapes were pinned from a real
    controller's Open API document before coding (`docs/DISCOVERY_DESIGN.md` §9).
    Live PoE *wattage* is deferred — the Open API exposes it only behind `PUT`
    config endpoints, not a clean read.
  New runtime deps: `zeroconf`, `defusedxml`. `config.yaml` gains
  `host_network: true` (multicast) + `discovery_service_types`.
- **Device onboarding (Phase 2 — discovered → Home Assistant).** The first
  *write* path: turns a discovered device into a managed HA device by driving
  the integration's **config flow**. HA's own discovery already pre-creates
  "discovered" flows; ASCIA surfaces them, correlates each to a discovered row
  (by `unique_id` → MAC → name), and completes them:
  - `GET /devices/onboarding/candidates` — pending flows + match + `one_click`
    (HA's `confirm_only`). `GET /devices/onboarding/{flow_id}` — a flow's
    current step (form fields). `POST /devices/onboarding/{flow_id}/submit` —
    empty body = one-click confirm; field values for credentialed steps.
  - **Slice 1 (one-click):** an "Add to HA" button on confirm-only matches.
    **Slice 2 (credentialed):** a dynamic, multi-step config-flow form (typed
    fields, per-field + base errors) for devices like ONVIF cameras.
  - OAuth/cloud flows stay "finish in HA" (out of scope). API shapes were
    captured from a live HA before coding (`docs/DISCOVERY_DESIGN.md` §10); the
    SDK gained `HAClient` config-flow methods + `ConfigFlow*` models.
- **Gateway auth.** Service-token bearer + signed `X-Ascia-User` header on
  every route, via the SDK's `ServiceAuthMiddleware`.
- **MQTT service discovery.** Uses the SDK's `SupervisorClient` to pull the
  broker's host/port/credentials from Supervisor at startup; degrades
  gracefully if no broker is installed (returns 502 on Zigbee routes with a
  structured error envelope, scene routes keep working).
- **OpenAPI contract.** At
  `../../../ascia-docs/api/ascia-device-manager.openapi.yaml`. Mirrors the
  shipped routes; validates via `openapi-spec-validator`.

### Verified

- **100+ add-on tests pass**, including HA WS reconnect under `MockHA`,
  Zigbee join/leave round-trip via `MockMQTT`, signed-auth gate, and the
  cache resync flow.
- **Discovery tested in three tiers:** Tier-1 unit (pure mappers/parsers + the
  WS-UsernameToken digest, recorded per-protocol fixtures), Tier-2 fake
  emulators (`tooling/compose` fake-mdns / fake-ssdp / fake-onvif / fake-omada)
  driven by a Linux CI integration job, and live-validated end to end against
  those emulators (mDNS, SSDP, ONVIF incl. authenticated SOAP → captured RTSP
  URL, and Omada token→sites→devices→clients→poe-ports).
- **Omada (W3) validated against a real controller** via
  `tooling/scripts/dev/omada_probe.py`: client-credentials auth + all four
  endpoints answered (`errorCode=0`, empty lists — no adopted hardware),
  confirming the auth flow incl. the token-request shape that wasn't in the
  captured spec.
- **Onboarding validated end to end against the real HAOS VM:** a discovered
  DLNA renderer was added to Home Assistant in one click from the ASCIA UI
  (candidates → match → confirm → config entry created → device flips to
  `in_ha`). Config-flow API shapes were captured first via
  `tooling/scripts/dev/ha_flows_probe.py`; the SDK config-flow methods,
  match/classify logic, and the slice-2 form are covered by unit tests.
- Running on a real **HAOS-in-UTM VM** end-to-end: built `aarch64`, pushed to
  private GHCR, published via the flat add-on repo, discovered by Supervisor,
  installed + started. Logs confirm `GET /core/api/states → 200` (HA access
  via the injected `SUPERVISOR_TOKEN`) and graceful MQTT degradation
  (`/services/mqtt → 400` with no broker installed).

## [0.0.0] — initial scaffolding

Add-on package created from the `ascia-base` template; no functional code
beyond the FastAPI shell and the SDK's `/health` route.

# Changelog — ascia-analytics

Energy/history analytics: consumption, top consumers, and the solar/grid/battery/house distribution
flow, from the add-on's own tiered time-series over HA power sensors. Format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versions use
[SemVer](https://semver.org/spec/v2.0.0.html).

## [0.2.0] - 2026-09-13

### Changed — BREAKING

- **`group_by=zone` → `group_by=area`** on `GET /analytics/consumption` and
  `/analytics/series`. The grouped key was always an HA `area_id`; the parameter
  now says so. The BFF is the only caller and ships in the same drop, so no alias
  was added — one would have bought nothing and cost a permanent exception in the
  vocabulary guard.

See `ascia-addons/docs/RENAME_AREA_AND_PLAN.md`.

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

## [Unreleased]

### Added

- **Initial add-on (0.1.0).** Ingests HA power-sensor `state_changed` (debounced) into a tiered
  SQLite store (`energy_sample` raw → `energy_hourly` rollup) behind a swappable `EnergyStore`
  interface. Classifies entities into energy roles (load/house/solar/grid/battery/battery_soc) so
  the whole-home meter + grid import/export + distribution flow are derived and reconcile.
- **Read API** (`prefix=/analytics`): `/energy/live` (instantaneous flow), `/energy/distribution`,
  `/energy/consumption?group_by=device|zone`, `/energy/top-consumers`, `/energy/summary`
  (with Δ vs previous period), and `/capabilities` (adaptive empty-states). `period` ∈
  today·week·month·year.
- **HA is the sensor bus only** — analytics owns storage + compute; never reads HA's recorder or
  another add-on's DB. Requires `ascia-addon-sdk` (HA WebSocket client + SQLite storage). Port 8128.

### Deferred (v1.1)

- Energy-counter (`device_class: energy`) ingestion; cost/tariff; overconsumption/standby alerts;
  gas/water; a `daily` rollup tier; the TimescaleDB `EnergyStore` swap.

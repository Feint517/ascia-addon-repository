# Changelog — ascia-analytics

Energy/history analytics: consumption, top consumers, and the solar/grid/battery/house distribution
flow, from the add-on's own tiered time-series over HA power sensors. Format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versions use
[SemVer](https://semver.org/spec/v2.0.0.html).

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

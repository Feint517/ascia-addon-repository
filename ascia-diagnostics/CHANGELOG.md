# Changelog — ascia-diagnostics

Appliance/system health: host metrics, per-container health, and a 0-100 score. Format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versions use
[SemVer](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- **Initial add-on (0.1.0).** Interval sampler + cached snapshot behind read routes
  (`/diagnostics/health|system|live|addons|report|history|capabilities`), including a cheap
  psutil-only `/live` for a fast (~2s) realtime dashboard poll. Collectors: host CPU/memory/
  temperature/uptime/load (psutil), disk (Supervisor `/host/info`), per-container health (Supervisor
  stats for every `ascia_*` add-on + HA Core + Supervisor), and best-effort network/backup peers.
- **Transparent health score** (`score.py`) — weighted, availability-renormalized mean with banded
  thresholds matching the spec's proactive-detection conditions; every non-ok signal yields an issue
  + recommendation. Pure and fully unit-tested.
- **Graceful degradation** everywhere: an unavailable metric (e.g. no thermal zone on a VM) is
  excluded from the score, and `/diagnostics/capabilities` reports what's available on this board.
- Requires `ascia-addon-sdk` ≥ 0.2.0 (`SupervisorClient` container/OS stats + `ContainerStats`).

### Deferred (v1.1)

- SSL-certificate expiry collector and the opt-in HA battery / house-temperature signals.
- Prometheus TSDB scraping (the `/metrics` seam already exists).

# Changelog — ascia-vpn

Secure remote access over a self-hosted NetBird (WireGuard) overlay. Format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versions use
[SemVer](https://semver.org/spec/v2.0.0.html).

This file is consumed by the Home Assistant add-on store when the add-on is
republished, and by anyone inspecting `/data` or the source tree on a running
add-on.

## [Unreleased]

> **Milestone (2026-06-10): deployed + validated on real HAOS.** v0.2.x enrolled
> against a live self-hosted NetBird stack, converged least privilege
> automatically, and the full product loop — family profile → QR → phone enrollment
> → reaching Home Assistant over the overlay IP → revoke — was driven through the
> `/vpn` dashboard. 244 tests green. Next: tier-3 (the same stack on a public HTTPS
> VPS, required for mobile/LTE clients) then the Slice 8 advanced-mode
> promote-or-delete checkpoint.

### Changed — v2 overlay architecture (2026-06-06/07)

- **Remote-access architecture replaced** (decision doc:
  `ascia-docs/design/ascia-vpn-remote-access.md`): the default mode now runs a
  NetBird agent dialing OUTBOUND to the ASCIA-operated management plane — zero
  exposed ports, no homeowner port-forward, CGNAT-proof. The v1 kernel-WG +
  port-forward path is parked behind `advanced_self_host` (env override
  `ASCIA_VPN_ADVANCED_SELF_HOST`) with a promote-or-delete checkpoint after
  tier-3 validation (roadmap Slice 8; default: delete).

### Added — v2 slices 1–5 (backend complete; 226 tests)

- `agent.py` — NetBird CLI wrapper (status parsing pinned to upstream source;
  Go zero-times, NeedsLogin quirks, setup key redacted from logs).
- Dual-mode entrypoint: mode-scoped lifespans, routers and readiness gates
  (`agent` gate + background connect-retry loop in agent mode).
- `management_client.py` — control-plane REST client (PAT `Token` scheme;
  setup keys with the **1-day minimum expiry** discovery, peers, groups;
  503 `management_unreachable` / 404 / 502 error mapping).
- `routes_overlay.py` — member profiles (per-profile NetBird groups as the
  peer↔profile join key; one-shot enrollment payloads, plaintext never
  persisted), support sessions, `/vpn/status` (degrades, never errors),
  7-row `/vpn/setup/check` incl. control-plane API ping + relayed-share
  advisory, `/vpn/audit` (v1-compatible contract).
- `sessions.py` — support-session reaper: hard 15/30/60-min cutoff via timed
  peer delete; survives restarts (state re-arm) and management outages
  (retry-until-confirmed).
- `overlay_watcher.py` — agent-status polling → audit entries (session
  start/end with direct-vs-relayed, 100 MB traffic markers, revocation-shaped
  "removed from network" ends).
- Tier-2 dev/validation kit at `tooling/compose/netbird-dev/` (pinned official
  NetBird stack, container peers, dev runner, interactive smoke). Full product
  loop validated against a live stack 2026-06-07 (smoke: 0 failed).
- OpenAPI contract: `ascia-docs/api/ascia-vpn.openapi.yaml` (mirrors the
  implemented overlay routes).

### Added — v2 Slice 7 (agent packaging; deployable image)

- **Bundled the NetBird agent in the image** (`Dockerfile`): the pinned binary
  (`NETBIRD_VERSION` ARG) is fetched per-arch (`TARGETARCH` → amd64/arm64) and
  **checksum-verified against NetBird's published `checksums.txt`** at build time.
  Runtime deps swapped from `wireguard-tools` to `iproute2`/`iptables`/`ca-certificates`.
- **Daemon S6 service** (`rootfs/etc/services.d/netbird/`): runs
  `netbird service run` with config persisted at **`/data/netbird/config.json`**, so
  the home keeps its overlay identity across image **updates** (only an
  uninstall-with-delete wipes it). On crash the daemon auto-restarts (2 s backoff)
  while the API service stays up and reports the overlay as down — a daemon blip no
  longer takes down the dashboard.
- `agent.py` threads `--config <path>` (env `ASCIA_VPN_NETBIRD_CONFIG`) through every
  daemon-touching command; secret redaction made position-independent.

### Changed

- **`config.yaml` for agent mode**: dropped the inbound `51820/udp` listener (the
  agent dials out — nothing to forward); caps trimmed to `NET_ADMIN` + `NET_RAW`
  (matches the upstream client container; BPF isn't in Supervisor's allow-list, so
  eBPF-only features degrade gracefully); added `setup_key` / `management_url` /
  `management_api_token` / `advanced_self_host` options + schema; version → 0.2.0.
- **Family tier is port-scoped** (`acl.py` `FAMILY_PORTS = ["8123", "443"]`): a family
  device reaches the home only on Home Assistant (8123) + HTTPS (443) over TCP — not
  ICMP, not other ports. Admin + technician remain all-ports. Tier-2 `acl-test`
  switched its positive probe from `ping` to a TCP/8123 check accordingly.

### Added — v1 slices 1–7 (now the parked advanced mode)

- Kernel-WireGuard implementation: `wg_interface.py` + conf/dump parsers,
  server `key_manager.py`, per-peer IP pool, wg-backed routes, connection
  watcher, packaging. Shipped as 0.1.x; retained unmodified behind
  `advanced_self_host` pending the Slice-8 decision.

## [0.0.0] — initial scaffolding

Add-on package directory created. No functional code beyond the manifest.

## [0.0.0] — initial scaffolding

Add-on package directory created. No functional code beyond the manifest.

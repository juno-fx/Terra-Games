# Conan Exiles Enhanced Server

Conan Exiles **Enhanced** (UE5) dedicated server workload powered by
[indifferentbroccoli/conan-exiles-enhanced-server-docker](https://github.com/indifferentbroccoli/conan-exiles-enhanced-server-docker).
Auto-installs the server files via DepotDownloader on first boot. Non-crossplay — Steam backend,
direct IP join.

> **License note**: Conan Exiles is a commercial game (Funcom). The dedicated server is free to
> run (installed via DepotDownloader), but every player must own the game.

**Type:** Workload Template (Server) — install the plugin in Terra, author it in Genesis,
users launch instances through Hubble.

## Image

`indifferentbroccoli/conan-exiles-enhanced-server-docker:latest` (`registry`/`repo`/`tag` fields).
**amd64 only** — the chart pins `nodeSelector: kubernetes.io/arch: amd64`.

## Ports

| Port | Protocol | Purpose | Exposed |
|------|----------|---------|---------|
| 7777 | UDP | Game traffic (`PORT`) — join endpoint | **Always** |
| 7778 | UDP | Pinger (`PORT` + 1) — the client queries it before connecting | **Always** |
| 27015 | UDP | Steam query (`QUERY_PORT`) — server browser | No (browser-only) |
| 25575 | TCP | RCON (not observed listening in the Enhanced build) | **Only when `rcon_password` is set** |

The pinger (7778) must be exposed because the Conan client queries it at **entered-port + 1**
before connecting — that's why the game + pinger are pinned to an adjacent nodePort pair (N/N+1).
The Steam query port (27015) stays unexposed: the browser pings *advertised* ports, which never
match NodePort mappings, so exposing it through the Service wouldn't help the browser anyway.

**RCON is gated on a password**: the image writes `RconEnabled=1`, but the UE5 Enhanced build was
observed NOT listening on 25575 in a live launch (connection refused) — RCON may not be available
in the Enhanced image at all. The exposure renders only when `rcon_password` is set (unset = not
exposed); revisit if upstream lands working RCON.

## Connecting

- **Join (NodePort, default)**: in-game "Join IP" → `<host-ip>:<game nodePort>`. The chart
  auto-derives an adjacent pair from the instance name (`game = N`, `pinger = N+1` — stable
  across syncs), so the client's +1 pinger query lands. Find the pair:
  `kubectl get svc <name> -n <namespace> -o jsonpath='{.spec.ports[*].nodePort}'`
  (e.g. `30500 30501` → join `192.168.70.25:30500`)
- **Collisions**: the auto-derived pair is deterministic — if k8s rejects the apply (ports already
  in use), rename the instance to re-derive
- **Join (LoadBalancer)**: pick `service_type: LoadBalancer` (k3s svclb opens the ports on every
  node IP) → join `<node-ip>:7777`, pinger auto-answers at 7778
- **Server browser visibility**: needs advertised ports reachable — LoadBalancer or hostPort;
  NodePort can't help the browser (advertised ports ≠ nodePorts), join-by-IP unaffected

## Launch Fields (Genesis workload authoring)

| Field | Default | Notes |
|-------|---------|-------|
| `server_name` | `Conan Exiles Enhanced Server` | Shown in the server browser |
| `server_password` | — | Stored in a Secret; blank = public server |
| `admin_password` | — | In-game admin password (Secret) |
| `rcon_password` | — | RCON password (Secret) — gates the 25575 TCP exposure (may not be listening in Enhanced — revisit upstream) |
| `max_players` | `40` | |
| `mods` | — | Comma-separated Steam Workshop IDs (e.g. `880454836,1159180273`) |
| `update_on_start` | `true` | `false` skips file validation on startup (faster restarts) |
| `cpu` / `memory` | `2` / `8Gi` | UE5 server is heavy — 8GB min, 16GB recommended |
| `cpuLimit` / `memoryLimit` | — / — | Pod limits (empty = none) |
| `service_type` | `NodePort` | NodePort or LoadBalancer (needs MetalLB/cloud LB) |
| `storage_class` | required | StorageClass for server files + saves |
| `storage_size` | `50` | Volume size in Gi (25GB min, 50GB recommended) |
| `restart_cron_schedule` | `0 4 * * *` | Cron schedule for auto-restart (daily at 4 AM). Set empty to disable. |

## Storage

Server files, saves and config live in `/home/steam/server-files` on the chart-rendered PVC
(`<name>-data`, subPath `conan-exiles`, `storage_class` + `storage_size` at launch).
Server runs as uid/gid 1000 (`fsGroup: 1000`).

## Known Issues

- First boot runs DepotDownloader (25-50GB download) — the startup probe allows 10 minutes;
  subsequent boots are fast unless `update_on_start` revalidates
- UE5 server is RAM-hungry: start at 8Gi, raise `memory` for large maps/mods
- RCON: image writes `RconEnabled=1` but the Enhanced build was observed not binding 25575 —
  the gated exposure is inert until upstream ships working RCON
- amd64-only image — arm64 nodes won't schedule this workload

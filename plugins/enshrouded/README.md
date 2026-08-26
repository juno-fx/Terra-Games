# Enshrouded Server

Enshrouded dedicated server workload powered by
[indifferentbroccoli/enshrouded-server-docker](https://github.com/indifferentbroccoli/enshrouded-server-docker).
The server binary is **Windows-only** and runs under Wine/Proton GE. Auto-installs via SteamCMD on
first boot. Non-crossplay — direct IP join.

> **License note**: Enshrouded is a commercial game (Keen Games). The dedicated server is free to
> run, but every player must own the game.

**Type:** Workload Template (Server) — install the plugin in Terra, author it in Genesis,
users launch instances through Hubble.

## Image

`indifferentbroccoli/enshrouded-server-docker` — `tag` field: `latest`, `wine` or `proton`.
**amd64 only** — the chart pins `nodeSelector: kubernetes.io/arch: amd64`.

## Ports

| Port | Protocol | Purpose | Exposed |
|------|----------|---------|---------|
| 15636 | UDP | Game traffic (the server's unpatched default) — join endpoint | **Always** |
| 15637 | UDP | Query (`QUERY_PORT`, patched into `enshrouded_server.json`) | **Always** |

The chart pins an **adjacent nodePort pair** (game N, query N+1) because the client queries the
server at entered-port + 1. Note: the upstream compose publishes **only** the query port (15637)
— that serves browser/discovery traffic; direct joins need the game port too, so this chart
exposes both (your call, and the valheim lesson).

## Connecting

- **Join**: in-game → join by IP → `<host-ip>:<game nodePort>`. The chart auto-derives the
  adjacent pair from the instance name (stable across syncs) — find it with
  `kubectl get svc <name> -n <namespace> -o jsonpath='{.spec.ports[*].nodePort}'`
  (e.g. `31234 31235` → join `192.168.70.25:31234`, query answers at 31235)
- **Collisions**: the auto-derived pair is deterministic — if k8s rejects the apply (ports already
  in use), rename the instance to re-derive
- **Password**: blank `server_password` = public server; set one at launch for a private server

## Launch Fields (Genesis workload authoring)

| Field | Default | Notes |
|-------|---------|-------|
| `tag` | `latest` | `latest`, `wine` or `proton` image variant |
| `server_name` | `Enshrouded Server` | Display name |
| `server_password` | — | Stored in a Secret; blank = public |
| `max_players` | `12` | |
| `update_on_start` | `true` | `false` skips file validation on startup (faster restarts) |
| `cpu` / `memory` | `2` / `8Gi` | 8GB min, 16GB recommended |
| `cpuLimit` / `memoryLimit` | — / — | Pod limits (empty = none) |
| `service_type` | `NodePort` | NodePort or LoadBalancer (needs MetalLB/cloud LB) |
| `storage_class` | required | StorageClass for server files + saves |
| `storage_size` | `40` | Volume size in Gi (20GB min, 40GB recommended) |
| `restart_cron_schedule` | `0 4 * * *` | Cron schedule for auto-restart (daily at 4 AM). Set empty to disable. |

## Storage

Server files, config and saves live in `/home/steam/enshrouded` on the chart-rendered PVC
(`<name>-data`, subPath `enshrouded`, `storage_class` + `storage_size` at launch).
Server runs as uid/gid 1000 (`fsGroup: 1000`).

## Known Issues

- First boot downloads the Windows server + wine setup (large) — the startup probe allows
  10 minutes; the server runs headless under Wine (`DISPLAY=:99`)
- Wine/Proton hosting adds startup latency vs native binaries; expect slower boots and higher
  CPU/RAM use than the game's Linux-native peers
- amd64-only image — arm64 nodes won't schedule this workload

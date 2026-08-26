# Satisfactory Server

Satisfactory dedicated server workload powered by
[indifferentbroccoli/satisfactory-server-docker](https://github.com/indifferentbroccoli/satisfactory-server-docker).
Auto-installs the server files via DepotDownloader on first boot. Non-crossplay — Steam backend,
direct IP join.

> **License note**: Satisfactory is a commercial game (Coffee Stain Studios). The dedicated server
> is free to run (installed via DepotDownloader), but every player must own the game.

**Type:** Workload Template (Server) — install the plugin in Terra, author it in Genesis,
users launch instances through Hubble.

## Image

`indifferentbroccoli/satisfactory-server-docker:latest` (`registry`/`repo`/`tag` fields).
**amd64 only** — the chart pins `nodeSelector: kubernetes.io/arch: amd64`.

## Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| N (auto-derived) | UDP + TCP | Game traffic (`GAME_PORT`) — join endpoint |
| N+1 (auto-derived) | TCP | Reliable channel (`RELIABLE_PORT`) |

The server **listens on its auto-derived nodePorts** — the env `GAME_PORT`/`RELIABLE_PORT` are
rendered from the derived pair, so the server's announced ports are always the reachable ones.
This is immune to whether the client derives the reliable channel as entered+1 or uses the
announced value. Join = `<host-ip>:<N>`, reliable channel answers at N+1.

## Connecting

- **Join**: in-game → "Join Game" → enter `<host-ip>:<game nodePort>`. The chart auto-derives the
  adjacent pair from the instance name (stable across syncs) — find it with
  `kubectl get svc <name> -n <namespace> -o jsonpath='{.spec.ports[*].nodePort}'`
  (e.g. `30409 30410` → join `192.168.70.25:30409`)
- **Collisions**: the auto-derived pair is deterministic — if k8s rejects the apply (ports already
  in use), rename the instance to re-derive
- **First join**: claim the server in-game (dedicated server tab → claim + set a strong password);
  the image's warning — shutdown via the server manager's console (`quit`) saves the world;
  SIGTERM is also handled gracefully by the image (30s grace in the chart)

## Launch Fields (Genesis workload authoring)

| Field | Default | Notes |
|-------|---------|-------|
| `max_players` | `4` | |
| `branch` | `public` | `public` (1.x stable) or `experimental` |
| `update_on_start` | `true` | `false` skips file validation on startup (faster restarts) |
| `auto_pause` | `true` | Pause when nobody is connected |
| `auto_save_on_disconnect` | `true` | Save when all players leave |
| `game_manifest` | — | Pin a SteamDB manifest ID for a specific version (requires Steam credentials) |
| `steam_username` / `steam_password` | — | Secret-backed; only used when `game_manifest` is set |
| `cpu` / `memory` | `2` / `12Gi` | 12GB min, 16GB recommended — heavy simulation |
| `cpuLimit` / `memoryLimit` | — / — | Pod limits (empty = none) |
| `service_type` | `NodePort` | NodePort or LoadBalancer (needs MetalLB/cloud LB) |
| `storage_class` | required | StorageClass for server files + saves |
| `storage_size` | `25` | Volume size in Gi (25GB minimum) |
| `restart_cron_schedule` | `0 4 * * *` | Cron schedule for auto-restart (daily at 4 AM). Set empty to disable. |

## Storage

Two mounts on the chart-rendered PVC (`<name>-data`):

- `satisfactory-files` → `/satisfactory` — server files (recreated on reinstall)
- `satisfactory-saves` → `.../FactoryGame/Saved/SaveGames` — **world saves** (precious)

Server runs as uid/gid 1000 (`fsGroup: 1000`).

## Known Issues

- First boot runs DepotDownloader (large download) — the startup probe allows 10 minutes
- The game port is UDP **and** TCP on 7777 — both must be open (the chart maps both)
- Version pinning via `game_manifest` requires Steam credentials and is a legal gray area for
  private servers — use at your own discretion
- amd64-only image — arm64 nodes won't schedule this workload

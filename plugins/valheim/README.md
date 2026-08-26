# Valheim Server

Valheim dedicated server workload powered by
[indifferentbroccoli/valheim-server-docker](https://github.com/indifferentbroccoli/valheim-server-docker).
Auto-installs the server via SteamCMD on first launch.

**Type:** Workload Template (Server) — install the plugin in Terra, author it in Genesis,
users launch instances through Hubble.

## Image

`indifferentbroccoli/valheim-server-docker:latest` (`registry`/`repo`/`tag` fields).

## Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 2456 | UDP | Game traffic (`PORT` env) — join endpoint |
| 2457 | UDP | Steam query (`PORT` + 1) — **join-critical**: the client's handshake queries it at entered-port + 1 |

The chart pins an **adjacent nodePort pair** (game N, query N+1) so the +1 query lands. No
ingress — Valheim traffic is pure UDP.

## Connecting

- **Join**: in-game "Join IP" → `<host-ip>:<game nodePort>`. The chart auto-derives the adjacent
  pair from the instance name (stable across syncs) — find it with
  `kubectl get svc <name> -n <namespace> -o jsonpath='{.spec.ports[*].nodePort}'`
  (e.g. `31710 31711` → join `192.168.70.25:31710`, query answers at 31711)
- **Collisions**: the auto-derived pair is deterministic — if k8s rejects the apply (ports already
  in use), rename the instance to re-derive
- **Server browser visibility**: needs the advertised port reachable — use `service_type:
  LoadBalancer` (k3s svclb opens it on every node IP) or hostPort; plain NodePort works for
  join-by-IP only

## Launch Fields (Genesis workload authoring)

| Field | Default | Notes |
|-------|---------|-------|
| `server_name` | `valheim` | Shown in the server list |
| `server_password` | — | **Required** — stored in a Secret |
| `world_name` | `dedicated` | Creates a new world or loads an existing one |
| `public` | `true` | `false` hides the server from the browser (join by IP) |
| `cpu` / `memory` | `2` / `4Gi` | Pod requests — raise memory for large worlds |
| `cpuLimit` / `memoryLimit` | — / — | Pod limits (empty = none) |
| `service_type` | `NodePort` | NodePort (high port on every node) or LoadBalancer (external IP — needs MetalLB/cloud LB) |
| `storage_class` | required | StorageClass for the world volume |
| `storage_size` | `10` | World volume size in Gi |
| `restart_cron_schedule` | `0 4 * * *` | Cron schedule for auto-restart (daily at 4 AM). Set empty to disable. |

Server runs as uid/gid 1000 (pod `fsGroup: 1000`) — the storage class must allow group write.

## Storage

The chart-rendered PVC (`<name>-data`) carries two subPaths:

- `valheim` → `/valheim-saves` — **world data** (precious)
- `valheim-files` → `/valheim` — server binaries (recreated on reinstall)

The `world_name` field picks the world; reusing the same name loads the existing save.

## Known Issues

- First launch runs SteamCMD (downloads the server) — up to 10 minutes; the startup
  probe allows for it (exec probe — UDP-only game, TCP probes can never succeed)
- Graceful shutdown is 30s — the world save completes on SIGTERM, don't shorten it
- BepInEx modding is supported by the image but not exposed as a field yet
  (add `BEPINEX_ENABLED` via env hints)

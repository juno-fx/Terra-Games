# Space Engineers Server

Space Engineers dedicated server workload via the
[Devidian docker-spaceengineers](https://github.com/Devidian/docker-spaceengineers) image —
the Windows dedicated server running under Wine on Debian 12.

> **License note**: Space Engineers is a commercial game. The dedicated server is free to
> run (installed via SteamCMD), but every player must own the game.

**Type:** Workload Template (Server) — install the plugin in Terra, author it in Genesis,
users launch instances through Hubble.

## Image

`devidian/spaceengineers` — `tag` field: `latest` (Wine 9.0) or `winestaging` (Wine 9.9).
~1.8 GB compressed.

## Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 27016 | UDP | Game traffic — the only exposed port |

Exposed via NodePort/LoadBalancer Service. No ingress — pure UDP.

## Connecting

- **Join**: in-game "Join IP" → `<host-ip>:<game nodePort>`. The chart auto-derives the nodePort
  from the instance name (stable across syncs) — find it with
  `kubectl get svc <name> -n <namespace> -o jsonpath='{.spec.ports[?(@.name=="game")].nodePort}'`
- **Collisions**: the auto-derived port is deterministic — if k8s rejects the apply (port already
  in use), rename the instance to re-derive

## World provisioning (required, manual)

This image does **not** generate worlds. The world must be created once and uploaded:

1. On a Windows machine, install **Space Engineers Dedicated Server** from Steam (Tools)
2. Use its setup tool to create and configure your world
3. Upload the resulting instance directory into the workload volume at
   `space-engineers/instances/<instance_name>/` (the PVC `<name>-data` — e.g. via
   `kubectl cp` into a temporary mount, or a file-browser sidecar)
4. Set the `instance_name` launch field to match the uploaded directory name
5. Launch the workload

The server downloads mods referenced by the world on first start.

## Launch Fields (Genesis workload authoring)

| Field | Default | Notes |
|-------|---------|-------|
| `instance_name` | `instance` | Must match the uploaded world directory |
| `tag` | `latest` | `latest` or `winestaging` |
| `public_ip` | — | Used by the image's own healthcheck; optional in Kubernetes (no probes) |
| `cpu` / `memory` | `2` / `4Gi` | Pod requests — Wine + server needs headroom |
| `cpuLimit` / `memoryLimit` | — / `8Gi` | Pod limits (empty = none) |
| `service_type` | `NodePort` | NodePort (high port on every node) or LoadBalancer (external IP — needs MetalLB/cloud LB) |
| `storage_class` | required | StorageClass for the world volume |
| `storage_size` | `20` | World volume size in Gi |
| `restart_cron_schedule` | `0 4 * * *` | Cron schedule for auto-restart (daily at 4 AM). Set empty to disable. |

## Storage

The chart-rendered PVC (`<name>-data`) backs four subPaths:

- `space-engineers/instances/` — world saves (precious)
- `space-engineers/plugins/` — Torch-style plugins (optional; added/removed by the entrypoint)
- `space-engineers/SpaceEngineersDedicated/` — server binaries (persisted so reinstalls are fast)
- `space-engineers/steam/` → `/root/.steam` — SteamCMD cache

## Known Issues

- No readiness/liveness probes (wine + UDP, no reliable TCP handshake) — crashes are
  recovered by the StatefulSet restart policy
- First launch is slow (SteamCMD install under Wine)
- Graceful shutdown 60s — the world save completes before the pod is killed
- VRage Remote Client connectivity is reported broken upstream (Devidian issue #36)
- A large world with many blocks eats RAM — raise `memory`/`memoryLimit` (defaults 4Gi/8Gi)

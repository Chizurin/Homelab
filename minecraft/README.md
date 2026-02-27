# Minecraft Server

Modded Minecraft server running on k3s, deployed via ArgoCD using the [itzg/minecraft-server](https://github.com/itzg/docker-minecraft-server) Helm chart.

## Overview

This follows the same GitOps pattern as the rest of the cluster:

1. Edit [`values.yaml`](values.yaml) to change any server setting or swap modpacks
2. Commit and push to `main`
3. ArgoCD detects the change and automatically resyncs — the server pod restarts with the new config

World data persists across restarts on a PersistentVolumeClaim (PVC).

## Prerequisites

- A [CurseForge API key](https://console.curseforge.com) (free — go to API Keys → Create key)
- The CurseForge page URL of the modpack you want to run
- Sealed Secrets controller running in the cluster (already set up)

## Initial Deployment

### 1. Create the namespace

The ArgoCD application creates the namespace automatically on first sync, but you need it to exist first in order to seal the secret against it:

```bash
kubectl create namespace minecraft
```

### 2. Seal your secrets

```bash
CF_API_KEY=your_cf_api_key
RCON_PASSWORD=your_rcon_password

kubectl create secret generic minecraft-secret \
  --namespace minecraft \
  --from-literal=CF_API_KEY="$CF_API_KEY" \
  --from-literal=rcon-password="$RCON_PASSWORD" \
  --dry-run=client -o yaml | \
  kubeseal --format yaml > secrets/minecraft/minecraft-secret.yaml
```

### 3. Set your modpack in values.yaml

Edit [`values.yaml`](values.yaml) and update `cfPageUrl` to your modpack's CurseForge page URL:

```yaml
cfPageUrl: "https://www.curseforge.com/minecraft/modpacks/all-the-mods-9"
```

### 4. Commit and push

```bash
git add apps/minecraft.yaml minecraft/values.yaml secrets/minecraft/minecraft-secret.yaml
git commit -m "Add Minecraft server"
git push
```

ArgoCD will pick it up within a minute and begin syncing. The first startup takes 5–15 minutes depending on modpack size (it downloads the modpack, installs mods, then starts the server).

### 5. Watch startup

```bash
kubectl logs -f -n minecraft $(kubectl get pod -n minecraft -o name | head -1)
```

The server is ready when you see: `Done! For help, type "help"`

## Connecting

Find your node's Tailscale hostname:

```bash
tailscale status --self
```

Connect from the Minecraft launcher: `<tailscale-hostname>:25565`

### Inviting Friends

Friends need Tailscale installed. Share access via the [Tailscale admin console](https://login.tailscale.com/admin/machines) → find your node → Share.

Once they're on your tailnet, they connect to the same address: `<tailscale-hostname>:25565`

## Changing Modpacks

1. Edit `cfPageUrl` (and optionally `type`, `version`) in [`values.yaml`](values.yaml)
2. Commit and push — ArgoCD restarts the server with the new modpack
3. The old world data stays on the PVC; the new modpack generates a fresh world

To keep a world, back it up first:

```bash
kubectl exec -n minecraft $(kubectl get pod -n minecraft -o name | head -1) \
  -- tar czf /tmp/world-backup.tar.gz /data/world
kubectl cp minecraft/$(kubectl get pod -n minecraft -o name | head -1 | cut -d/ -f2):/tmp/world-backup.tar.gz ./world-backup.tar.gz
```

## Common Commands

| Task | Command |
|------|---------|
| Watch server logs | `kubectl logs -f -n minecraft $(kubectl get pod -n minecraft -o name \| head -1)` |
| Open server console (RCON) | `kubectl exec -it -n minecraft $(kubectl get pod -n minecraft -o name \| head -1) -- rcon-cli` |
| Force restart server | `kubectl delete pod -n minecraft $(kubectl get pod -n minecraft -o name \| head -1)` |
| Check storage | `kubectl get pvc -n minecraft` |
| Check pod status | `kubectl get pods -n minecraft` |
| Check all resources | `kubectl get all -n minecraft` |

## Configuration Reference

All settings live in [`values.yaml`](values.yaml). Key sections:

| Setting | Description |
|---------|-------------|
| `minecraftServer.type` | Server type: `AUTO_CURSEFORGE`, `MODRINTH`, `FORGE`, `FABRIC`, `NEOFORGE`, `VANILLA` |
| `minecraftServer.version` | Minecraft version (`LATEST` or e.g. `1.20.1`) |
| `minecraftServer.cfPageUrl` | CurseForge modpack page URL |
| `minecraftServer.memory` | Java heap size (e.g. `4096M`) |
| `minecraftServer.difficulty` | `peaceful`, `easy`, `normal`, `hard` |
| `minecraftServer.gameMode` | `survival`, `creative`, `adventure`, `spectator` |
| `minecraftServer.maxPlayers` | Maximum concurrent players |
| `minecraftServer.viewDistance` | Render distance in chunks |
| `minecraftServer.ops` | Comma-separated operator usernames |
| `minecraftServer.whitelist` | Comma-separated whitelist usernames |
| `minecraftServer.levelSeed` | World seed (`""` = random) |
| `nodePort` | Port friends connect to (default `25565`) |
| `persistence.dataDir.Size` | PVC size for world data and mods |
| `resources.limits.memory` | Kubernetes memory limit (keep above `memory` setting) |

## Troubleshooting

### Pod is stuck in `Pending`

Check storage and node resources:

```bash
kubectl describe pod -n minecraft $(kubectl get pod -n minecraft -o name | head -1)
kubectl get pvc -n minecraft
kubectl describe nodes
```

Common causes: not enough CPU/RAM on the node, or PVC failed to provision.

### Pod keeps crash-looping

Read the logs immediately after a crash:

```bash
kubectl logs -n minecraft $(kubectl get pod -n minecraft -o name | head -1) --previous
```

Common causes:
- **OOM (out of memory)**: increase `resources.limits.memory` (must be higher than `minecraftServer.memory`)
- **Bad API key**: check the `minecraft-secret` secret exists and has the correct key
- **EULA not accepted**: ensure `eula: "TRUE"` is set in values.yaml

### Modpack won't download

Verify the secret exists and has the right key:

```bash
kubectl get secret minecraft-secret -n minecraft
kubectl describe secret minecraft-secret -n minecraft
```

Verify the `cfPageUrl` is a valid, publicly accessible CurseForge modpack page (not a private or deleted pack).

### Friends can't connect

1. Confirm the friend has Tailscale installed and is on your tailnet
2. Confirm the server pod is `Running` and shows `Done!` in logs
3. Confirm the NodePort is correct: `kubectl get svc -n minecraft`
4. Test from your own machine first using the Tailscale hostname

### Server is lagging

Lower the render/simulation distance and check resource usage:

```bash
kubectl top pod -n minecraft
```

Adjustments to make in `values.yaml`:
- Decrease `viewDistance` (try `8` or `6`)
- Decrease `simulationDistance` (try `6`)
- Increase `minecraftServer.memory` (and raise `resources.limits.memory` to match)

### World disappeared after pod restart

The world data should persist on the PVC. Verify it is bound:

```bash
kubectl get pvc -n minecraft
```

Status must be `Bound`. If it shows `Lost` or `Pending`, the underlying storage volume may have an issue. Check:

```bash
kubectl describe pvc -n minecraft
```

### Pod restarts before server finishes loading

The liveness probe is killing the pod before the modpack finishes starting up. Increase the probe delay in `values.yaml`:

```yaml
livenessProbe:
  initialDelaySeconds: 300  # increase from 120
readinessProbe:
  initialDelaySeconds: 300
```

### Tailscale operator / networking issues

Since Tailscale runs on the host (not inside k3s), the server is accessible on the node's Tailscale IP via the NodePort. If `<tailscale-hostname>:25565` is unreachable:

```bash
# Confirm the NodePort service is up
kubectl get svc -n minecraft

# Check which port is assigned
kubectl get svc -n minecraft -o jsonpath='{.items[0].spec.ports[0].nodePort}'

# Confirm Tailscale is connected
tailscale status
```

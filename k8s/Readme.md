# Kubernetes

This folder contains a sample **StatefulSet** deploy for a single Cronicle manager using the default **Filesystem** storage engine.

File:

- `statefulset.yaml` — Namespace, Secret, headless + ClusterIP Services, and StatefulSet with a PVC for `/opt/cronicle/data`

A StatefulSet is preferred over a plain Deployment because Cronicle persists the manager hostname (and IP) under `global/servers` on first setup. Stable pod identity (`cronicle-0`) plus a durable data volume keeps that registration valid across restarts, reschedules, and upgrades.

# Prerequisites

- A working Kubernetes cluster (`kubectl` configured)
- A default `StorageClass` (or set `storageClassName` on the PVC template)
- Ability to pull the image referenced in `statefulset.yaml`

# What gets created

| Resource | Name | Purpose |
|---|---|---|
| Namespace | `cronicle` | Isolates the install |
| Secret | `cronicle-secret` | `secret_key` (and optional `default_api_key`) |
| Service (headless) | `cronicle-headless` | Stable DNS for the StatefulSet pod |
| Service (ClusterIP) | `cronicle` | UI / ingress backend |
| StatefulSet | `cronicle` | 1 manager replica |
| PVC (via template) | `data-cronicle-0` | Persists `/opt/cronicle/data` |

# Before you apply

Edit `statefulset.yaml` and set a real secret key. It must contain at least one letter or special character (not purely numeric):

```yaml
stringData:
  secret_key: "MyCronSecretKey"
  default_api_key: ""   # optional; leave empty if unused
```

Optionally adjust:

- **image** — pin a release tag instead of whatever is in the file
- **CRONICLE_base_app_url** — should match how browsers/clients reach Cronicle
- **storage** size / `storageClassName` under `volumeClaimTemplates`
- **TZ**, resource requests/limits
- **CRONICLE_setup=minimal** — uncomment to skip demo setup content on first init

# Deploy

```bash
kubectl apply -f k8s/statefulset.yaml
```

Wait for the pod:

```bash
kubectl -n cronicle rollout status statefulset/cronicle
kubectl -n cronicle get pods,svc,pvc
```

# Access the UI

Port-forward (simplest for local/dev):

```bash
kubectl -n cronicle port-forward svc/cronicle 3012:80
# open http://localhost:3012
# default login: admin / admin
```

The ClusterIP service listens on port **80** and targets container port **3012**. If you change the Service port mapping, update the port-forward accordingly.

For production, put an Ingress (or LoadBalancer) in front of `svc/cronicle` and set `CRONICLE_base_app_url` to the public URL.

# Design notes (why these settings)

## Stable hostname

The pod is pinned as `cronicle-0` via:

- StatefulSet ordinal name
- `spec.template.spec.hostname: cronicle-0`
- `HOSTNAME=cronicle-0` env (used when writing manager identity into storage)

Always keep the same hostname for a given data volume. If you rename the host while reusing old data, manager eligibility can fail until identity is healed/reset.

## Persistent data only

Same rule as Docker: you mainly need to persist the **data** folder.

| Path | Volume | Notes |
|---|---|---|
| `/opt/cronicle/data` | PVC (`data-cronicle-0`) | Schedules, users, job history, server list — **required** |
| `/opt/cronicle/logs` | `emptyDir` | Ephemeral |
| `/opt/cronicle/queue` | `emptyDir` | Ephemeral |

## Manager entrypoint

```yaml
args: ["manager"]
```

The image `ENTRYPOINT` is `tini`. `args` runs the `manager` script, which:

1. Runs storage setup on first boot (if needed)
2. Starts Cronicle in manager mode (`--manager`)

Do **not** scale this StatefulSet above **1** replica while using local Filesystem storage. Each pod would get its own PVC and its own cluster identity. For multi-node clusters use a shared storage engine (S3/SQL) and separate worker workloads.

## Secret key

```yaml
env:
  - name: CRONICLE_secret_key
    valueFrom:
      secretKeyRef:
        name: cronicle-secret
        key: secret_key
```

All nodes in a cluster must share the same secret. Prefer creating/updating the Secret out of band rather than committing real keys:

```bash
kubectl -n cronicle create secret generic cronicle-secret \
  --from-literal=secret_key='MyCronSecretKey' \
  --from-literal=default_api_key='' \
  --dry-run=client -o yaml | kubectl apply -f -
```

# Logs and debugging

```bash
# pod status / events
kubectl -n cronicle describe pod cronicle-0

# application logs
kubectl -n cronicle logs -f cronicle-0

# previous crash (if restarted)
kubectl -n cronicle logs cronicle-0 --previous
```

Shell into the pod if needed:

```bash
kubectl -n cronicle exec -it cronicle-0 -- bash
```

# Restart / upgrade

Force a rolling restart (keeps the PVC):

```bash
kubectl -n cronicle rollout restart statefulset/cronicle
kubectl -n cronicle rollout status statefulset/cronicle
```

Upgrade the image:

```bash
kubectl -n cronicle set image statefulset/cronicle \
  cronicle=ghcr.io/ayomide101/cronicle-edge:v1.14.7

# or edit the manifest and re-apply
kubectl apply -f k8s/statefulset.yaml
```

# Tear down

Delete the workload but **keep** data (PVC retained by default when the StatefulSet is deleted):

```bash
kubectl -n cronicle delete statefulset cronicle
kubectl -n cronicle delete svc cronicle cronicle-headless
```

Delete everything including data:

```bash
kubectl delete -f k8s/statefulset.yaml
kubectl -n cronicle delete pvc data-cronicle-0
# optional: remove namespace
kubectl delete namespace cronicle
```

# Workers (optional)

Worker nodes are stateless and do not need a data PVC. They must:

- Use the **same** `CRONICLE_secret_key`
- Reach the manager over the cluster network
- Be added in the UI under **Admin → Servers** (hostname must match the worker pod/DNS name)

With Filesystem storage on the manager only, workers still need the manager to contact them (Socket.IO). Prefer hostnames that resolve inside the cluster, and keep `server_comm_use_hostnames` enabled (default).

For a real multi-manager or HA setup, switch storage to S3/SQL first; do not simply set `replicas: 2` on this StatefulSet.

# Ingress sketch

Example (adjust host, class, and TLS to your cluster):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cronicle
  namespace: cronicle
spec:
  rules:
    - host: cronicle.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: cronicle
                port:
                  number: 80
```

After exposing a public URL, update:

```yaml
- name: CRONICLE_base_app_url
  value: "https://cronicle.example.com"
```

# Related

- Docker examples and swarm notes: [Docker/Readme.md](../Docker/Readme.md)
- Main project install notes: [Readme.md](../Readme.md)

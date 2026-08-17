# AGENTS.md

This repository contains **plain Kubernetes manifests** (no Helm, no Kustomize, no CI) for self-hosted services deployed to a home MicroK8s cluster. Each top-level directory is one self-contained service.

## Essential Commands

There is no build/test/lint pipeline. Everything is manual `kubectl apply`. Apply order matters — namespace first, then config/secrets/storage, then workloads:

```bash
# Example for a service (replace <service> with pihole or n8n)
kubectl apply -f <service>/namespace.yaml
kubectl apply -f <service>/secret.yaml        # fill placeholders first!
kubectl apply -f <service>/configmap.yaml      # if present
kubectl apply -f <service>/pvc.yaml
kubectl apply -f <service>/service.yaml
kubectl apply -f <service>/deployment.yaml     # or statefulset.yaml
# For n8n, also apply postgres/ and runner/ subdirectories (same ordering)
```

Verify with `kubectl -n <namespace> get pods,svc,pvc`.

## Cluster Assumptions (non-obvious)

These are not documented anywhere except inline in the YAML. They must hold or the manifests will not schedule correctly:

- **MicroK8s** is the target cluster. All PVCs pin `storageClassName: microk8s-hostpath`. Change this if targeting a different provisioner.
- **MetalLB** must be installed — every externally-exposed Service is `type: LoadBalancer` and pins a specific LAN IP via annotation.
- The **LAN subnet is `192.168.29.0/24`**. LoadBalancer IPs are hard-coded into this range (pihole `.129`, n8n `.135`, postgres `.134`).
- There is **no ingress controller** in use; services are reached directly at their LoadBalancer IP.

## Repository Layout & Conventions

- One directory per service (`pihole/`, `n8n/`). Subcomponents of a service live in nested dirs (e.g. `n8n/postgres/`, `n8n/runner/`) and are applied independently.
- Canonical file set per service: `namespace.yaml`, `deployment.yaml` (or `statefulset.yaml`), `service.yaml`, `pvc.yaml`, `secret.yaml` (or `secrets.yaml`), `configmap.yaml` (optional).
- Resources are named after the service; PVCs follow `<service>-pvc` or `<service>-etc`. Labels are uniformly `app: <service-name>` and selectors match on that single label.
- YAML is 2-space indented. Inline `#` comments are used to flag values that must be customized (timezone, IPs, passwords).

## Secrets Handling (important)

Secrets are **plain Kubernetes Secrets with placeholder values** that must be replaced before applying. There is no SealedSecrets / ExternalSecrets / SOPS layer.

- `pihole/secret.yaml` uses `stringData:` with `FTLCONF_webserver_api_password: <PASSWORD>`.
- `n8n/secrets.yaml` uses base64 `data:` with placeholders `<DB_NONROOT_PASSWORD>`, `<DB_NONROOT_USER>`, `<DB_ROOT_PASSWORD>`, `<DB_USER>`, `<ENCRYPTION_KEY>`, `<N8N_RUNNERS_AUTH_TOKEN>`. These must be base64-encoded before applying (or convert to `stringData:`).
- The n8n `encryption-key` is critical — losing it makes existing workflow credentials undecryptable.

## Gotchas & Inconsistencies

These are easy to miss when editing a single file:

1. **Inconsistent MetalLB annotation key.** `pihole/service.yaml` uses `metallb.io/loadBalancerIPs` (plural); `n8n/service.yaml` and `n8n/postgres/service.yaml` use `metallb.io/loadBalancerIP` (singular). Both work in MetalLB, but be deliberate about which you use.

2. **n8n resources omit the `namespace:` field.** `pihole/` sets `namespace: pihole` on every resource; most `n8n/` manifests (deployment, service, pvc, secrets, configmap, and everything under `postgres/` and `runner/`) do **not**. They will land in the namespace implied by `kubectl`'s current context (typically `default`) unless applied with `-n n8n`. Apply with `kubectl -n n8n apply -f ...` or add the field explicitly.

3. **`postgres/statefulset.yaml` references `serviceName: postgres-headless`, but no Service named `postgres-headless` is defined anywhere** — only `postgres-svc` exists. This is a latent issue; Kubernetes will still create the StatefulSet but the headless DNS record it expects will not exist. If you need stable pod DNS, add a headless Service named `postgres-headless`.

4. **The Postgres StatefulSet does not use `volumeClaimTemplates`.** It mounts a pre-created PVC (`postgres-pvc`) via `volumes:`, so it behaves like a Deployment storage-wise. Editing replica count > 1 would cause an RWO PVC attach conflict.

5. **Two different image registries for n8n components.** The main n8n Deployment pulls `docker.n8n.io/n8nio/n8n:stable`; the runner pulls `n8nio/runners:stable` (Docker Hub). Don't assume one registry.

6. **Both the n8n broker port (5679) and Postgres (5432) are exposed as `type: LoadBalancer` on the LAN.** This is how the manifests are written — if you want internal-only traffic for the broker or DB, change these to `ClusterIP`. The runner reaches the broker via the in-cluster DNS name `http://n8n:5679`, so the LoadBalancer exposure of 5679 is not required for runner functionality.

7. **`N8N_HOST` is set to `n8n.dhairya.co`** with a commented-out IP alternative. This affects callback URLs and webhook URLs n8n generates; change it to match your actual hostname/IP.

## n8n Architecture Notes

n8n is deployed in **external runners mode** (`N8N_RUNNERS_MODE=external`), which splits execution from the main process:

- **n8n main** (`deployment.yaml`): serves UI/API on port 5678 and a task broker on 5679. Stores data in Postgres, persists files to a 10Gi PVC at `/home/node/.n8n`.
- **n8n-runner** (`runner/deployment.yaml`): a separate Deployment (image `n8nio/runners:stable`) that connects to the broker at `http://n8n:5679` using a shared `N8N_RUNNERS_AUTH_TOKEN` from `n8n-secrets`. It is the component that actually executes workflow nodes.
- **postgres** (`postgres/`): a StatefulSet running `postgres:18`. An init script (`postgres/init-script.yaml`, mounted at `/docker-entrypoint-initdb.d`) creates a **non-root user** that n8n uses (the main `POSTGRES_USER` is the root/admin). n8n's `DB_POSTGRESDB_USER`/`PASSWORD` reference the non-root secret keys, while Postgres's `POSTGRES_USER`/`POSTGRES_PASSWORD` reference the root keys. Keep these two pairs straight when editing secrets.

The runner is horizontally scalable; the main n8n Deployment and Postgres are not (single replica, RWO PVCs).

## pihole Architecture Notes

- **`replicas: 1` is mandatory.** Port 53 is bound via `hostPort`, so two pods on one node would conflict. The Deployment uses `strategy: Recreate` to avoid running two pods simultaneously during rollouts. Do not raise the replica count.
- DNS (port 53, TCP+UDP) is exposed via `hostPort` directly on the node's IP; the web UI (80/443) is exposed via the LoadBalancer Service. This is a deliberately mixed exposure model.
- `externalTrafficPolicy: Local` is set on the LoadBalancer.
- The container requires capabilities `NET_ADMIN`, `SYS_TIME`, `SYS_NICE`.
- Timezone defaults to `Asia/Kolkata` — change `TZ` if deploying elsewhere.

## Editing Guidance

- When adding a new service, follow the `pihole/` layout (it is the most complete and consistent example, including a README). Mirror its file set and apply the `namespace:` field to every resource.
- Preserve the inline comments that mark customizable values — they are the only documentation of "change this before deploying."
- There is no validation step beyond `kubectl apply`. Consider `kubectl apply --dry-run=server -f <file>` to catch errors before hitting the cluster.
- This is not currently a git repository; there is no commit/PR workflow to follow.

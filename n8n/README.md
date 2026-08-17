# n8n Kubernetes Deployment

This directory contains Kubernetes manifests to deploy [n8n](https://n8n.io/) — a workflow automation tool — backed by Postgres and running in **external task runners mode**.

## Architecture

Three components work together:

- **n8n main** (`deployment.yaml`): serves the UI/API on port `5678` and a task broker on port `5679`. Persists its files to a 10Gi PVC and stores workflow data in Postgres.
- **Postgres** (`postgres/`): a StatefulSet running `postgres:18`. An init script creates a dedicated non-root database user for n8n.
- **n8n-runner** (`runner/`): a separate Deployment (image `n8nio/runners:stable`) that connects to the main instance's broker and actually executes workflow nodes. Scalable independently.

## Prerequisites

- A running Kubernetes cluster (e.g., MicroK8s).
- [MetalLB](https://metallb.universe.tf/) installed and configured (required to fulfill the LoadBalancer Service IPs).
- A StorageClass named `microk8s-hostpath` available for persistent storage (or modify the PVCs to match your cluster's storage provisioner).

## Manifests Overview

- **`namespace.yaml`**: Creates a dedicated namespace `n8n`.
- **`configmap.yaml`**: Non-sensitive n8n configuration (`node-functions-allow-external: cheerio`).
- **`secrets.yaml`**: Database credentials (root and non-root), the n8n encryption key, and the runners auth token. All values are base64-encoded placeholders (`<...>`). **Replace all of them before deploying!**
- **`pvc.yaml`**: Provisions a 10Gi PersistentVolumeClaim (`n8n-pvc`) for n8n's data directory (`/home/node/.n8n`).
- **`deployment.yaml`**: Deploys n8n with `N8N_RUNNERS_MODE=external`. Exposes ports `5678` (UI/API) and `5679` (task broker).
- **`service.yaml`**: LoadBalancer service pinning IP `192.168.29.135`, exposing both `5678` and `5679`.
- **`postgres/`**: Postgres 18 backend.
  - `statefulset.yaml`: single replica, mounts the pre-created `postgres-pvc` and the init script.
  - `init-script.yaml`: ConfigMap with a shell script (mounted at `/docker-entrypoint-initdb.d`) that creates the non-root user n8n connects with and grants it privileges.
  - `configmap.yaml`: `POSTGRES_DB: n8n_db`.
  - `pvc.yaml`: 15Gi PersistentVolumeClaim for database storage.
  - `service.yaml`: LoadBalancer service pinning IP `192.168.29.134`, exposing port `5432`.
- **`runner/`**: External task runners.
  - `deployment.yaml`: connects to the broker at `http://n8n:5679` using the shared `N8N_RUNNERS_AUTH_TOKEN`.
  - `service.yaml`: ClusterIP service (port `5678`) for the runners.

## Deployment Instructions

1. **Fill in the secrets**: Open `secrets.yaml` and replace every `<...>` placeholder with a base64-encoded value (`echo -n 'value' | base64`). Alternatively, convert `data:` to `stringData:` and use plain values. Keep these keys straight:
   - `db-user` / `db-root-password`: the Postgres **superuser** (used by the `postgres` container itself).
   - `db-nonroot-user` / `db-nonroot-password`: the **application user** n8n connects with (created by the init script).
   - `encryption-key`: n8n's `N8N_ENCRYPTION_KEY`. **Back this up** — losing it makes all stored workflow credentials undecryptable.
   - `n8n-runners-auth-token`: shared secret between the main instance and the runners.
2. **Update the host/IP**: In `deployment.yaml`, set `N8N_HOST` to the hostname or IP users will reach n8n at (currently `n8n.dhairya.co`). This affects generated webhook and callback URLs. Update the MetalLB IPs in `service.yaml` and `postgres/service.yaml` if they don't fit your network.
3. **Apply the manifests**, namespace first:

   ```bash
   kubectl apply -f namespace.yaml
   ```

   Then apply everything else into the `n8n` namespace. **Note**: the manifests in this directory (and its subdirectories) do not set an explicit `namespace:` field, so you must pass `-n n8n` or they will land in the `default` namespace:

   ```bash
   kubectl -n n8n apply -f configmap.yaml
   kubectl -n n8n apply -f secrets.yaml
   kubectl -n n8n apply -f pvc.yaml
   kubectl -n n8n apply -f postgres/
   kubectl -n n8n apply -f service.yaml
   kubectl -n n8n apply -f deployment.yaml
   kubectl -n n8n apply -f runner/
   ```

4. **Verify**:

   ```bash
   kubectl -n n8n get pods,svc,pvc
   ```

   Wait until the `postgres` pod is ready before the `n8n` pod will start successfully (the database must exist and the init script must have run).

## Accessing n8n

Once the pod is running and ready, access the UI at:

`http://192.168.29.135:5678` (or the `N8N_HOST` / IP you configured).

## Important Notes

- **External runners mode**: `N8N_RUNNERS_MODE=external` means the main n8n process does not execute workflow nodes itself — the `n8n-runner` pods do, via the broker on port `5679`. If the runners aren't running or can't reach the broker, workflows will hang.
- **Image registries differ**: the main image is `docker.n8n.io/n8nio/n8n:stable`, while the runner image is `n8nio/runners:stable` (Docker Hub). Both must be pullable from your cluster.
- **Port 5679 and Postgres are LAN-exposed**: both the n8n Service (broker port `5679`) and `postgres-svc` (`5432`) are `type: LoadBalancer`, meaning they're reachable from your LAN — not just inside the cluster. If you want them internal-only, change those services to `ClusterIP`. The runners use the in-cluster DNS name `n8n:5679`, so they don't need the LoadBalancer exposure.
- **Scaling**: the runner Deployment can be scaled up freely. The main n8n Deployment and Postgres StatefulSet are single-replica only (they share ReadWriteOnce PVCs).
- **Headless service**: the Postgres StatefulSet declares `serviceName: postgres-headless`, but no such Service is defined here (only `postgres-svc`). Add a headless Service with that name if you need stable per-pod DNS records.

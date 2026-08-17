# Self-Hosted Services

Kubernetes manifests for running self-hosted services on a home [MicroK8s](https://microk8s.io/) cluster. Each top-level directory is a self-contained service with its own namespace and manifests — plain YAML, no Helm or Kustomize.

## Services

| Service | Description | Namespace | Access |
|---|---|---|---|
| [`pihole/`](./pihole) | DNS sinkhole (ad-blocking DNS for the LAN) | `pihole` | `http://192.168.29.129/admin` |
| [`n8n/`](./n8n) | Workflow automation, with Postgres backend and external task runners | `n8n` | `http://192.168.29.135:5678` |

Each service has its own README with detailed deployment instructions.

## Cluster Prerequisites

These manifests assume:

- **MicroK8s** — all PVCs use `storageClassName: microk8s-hostpath`. Change this to match your cluster's storage provisioner if different.
- **[MetalLB](https://metallb.universe.tf/)** — externally exposed services are `type: LoadBalancer` with pinned LAN IPs via MetalLB annotations.
- **LAN subnet `192.168.29.0/24`** — LoadBalancer IPs are hard-coded into this range. Update the `metallb.io/loadBalancerIP[s]` annotations if your network differs.

## Repository Layout

```
self-host/
├── pihole/            # DNS sinkhole
│   ├── namespace.yaml
│   ├── secret.yaml
│   ├── pvc.yaml
│   ├── service.yaml
│   ├── deployment.yaml
│   └── README.md
└── n8n/               # Workflow automation
    ├── namespace.yaml
    ├── configmap.yaml
    ├── secrets.yaml
    ├── pvc.yaml
    ├── service.yaml
    ├── deployment.yaml
    ├── postgres/      # Postgres 18 backend (StatefulSet)
    ├── runner/        # External task runners
    └── README.md
```

## Deployment

Apply order matters: namespace first, then secrets/configmaps/storage, then services, then workloads. See each service's README for exact commands.

```bash
# General pattern per service
kubectl apply -f <service>/namespace.yaml
kubectl -n <service> apply -f <service>/
```

**Important**: secrets contain placeholder values (e.g. `<PASSWORD>`, `<ENCRYPTION_KEY>`). Replace them before applying, and never commit real credentials.

## Notes for Contributors & Agents

See [AGENTS.md](./AGENTS.md) for non-obvious conventions and gotchas (MetalLB annotation inconsistency, namespace handling differences between services, the external-runners architecture, and more).

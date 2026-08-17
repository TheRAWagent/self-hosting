# Pi-hole Kubernetes Deployment

This repository contains Kubernetes manifests to deploy [Pi-hole](https://pi-hole.net/) as a DNS sinkhole on your cluster.

## Prerequisites

- A running Kubernetes cluster (e.g., MicroK8s).
- [MetalLB](https://metallb.universe.tf/) installed and configured (required to fulfill the LoadBalancer Service IP).
- A StorageClass named `microk8s-hostpath` available for persistent storage (or modify `pvc.yaml` to match your cluster's storage provisioner).

## Manifests Overview

- **`namespace.yaml`**: Creates a dedicated namespace `pihole` for all resources.
- **`secret.yaml`**: Defines a Kubernetes Secret containing the Web UI admin password. The default is `PASSWORD`. **Change this before deploying!**
- **`pvc.yaml`**: Provisions a 1Gi PersistentVolumeClaim (`pihole-etc`) for Pi-hole's configuration data (`/etc/pihole`).
- **`deployment.yaml`**: Deploys a single replica of Pi-hole.
  - Exposes port 53 directly to the host network via `hostPort` for reliable DNS resolution.
  - Sets the timezone to `Asia/Kolkata`.
  - Configures readiness and liveness probes.
  - Requests necessary capabilities (`NET_ADMIN`, `SYS_TIME`, `SYS_NICE`).
- **`service.yaml`**: Creates a LoadBalancer service to expose the DNS (53 UDP/TCP) and Web UI (80, 443 TCP) ports. It specifically requests the IP `192.168.29.129` via a MetalLB annotation.

## Deployment Instructions

1. **Update the Password**: Open `secret.yaml` and change the `FTLCONF_webserver_api_password` value to a secure password.
2. **Update Service IP**: Open `service.yaml` and change the `metallb.io/loadBalancerIPs` annotation to an available IP in your MetalLB pool, if `192.168.29.129` is not applicable to your network.
3. **Update Timezone**: Open `deployment.yaml` and update the `TZ` environment variable if you are not in the `Asia/Kolkata` timezone.
4. **Apply the manifests**:

   Apply the namespace first:
   ```bash
   kubectl apply -f namespace.yaml
   ```

   Apply the remaining manifests:
   ```bash
   kubectl apply -f secret.yaml
   kubectl apply -f pvc.yaml
   kubectl apply -f service.yaml
   kubectl apply -f deployment.yaml
   ```

## Accessing the Web UI

Once the pod is running and ready, you can access the Pi-hole Web UI at:

`http://192.168.29.129/admin` (or the IP you configured in `service.yaml`).

Log in using the password defined in your `secret.yaml`.

## Important Notes

- **Replicas**: The deployment is strictly set to 1 replica with a `Recreate` deployment strategy. Scaling beyond 1 is not supported because `hostPort: 53` is used, which would cause port binding conflicts if two pods landed on the same node.
- **hostPort**: The `hostPort` configuration for port 53 ensures that devices on your local network can reliably reach Pi-hole directly using the node's IP address.

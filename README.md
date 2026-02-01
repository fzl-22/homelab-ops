# Homelab Operations (homelab-ops)

This repository serves as the **Single Source of Truth** for the core infrastructure of the ASUS Homelab. It follows **GitOps** principles to manage networking, namespaces, and cluster-wide configurations.

> **Note:** Application-specific workloads live in their own repositories. This repo is for the *platform* itself.

## Repository Structure

```text
homelab-ops/
├── core/
│   ├── network/
│   │   └── tunnel.yaml      # Cloudflare Tunnel (The Gateway)
│   └── namespaces/          # Namespace definitions
├── scripts/                 # Utility scripts (install-k3s, etc.)
└── README.md
```

## Architecture

- **Cluster**: K3s on Ubuntu Server installed on my old ASUS laptop.
- **Ingress**: Cloudflare Tunnel (running as a Deployment).
- **Strategy**: No open ports. The cluster dials out to Cloudflare to receive traffic.

## Prerequisites

- **Kubernetes Cluster** (K3s recommended).
- **Cloudflare Account** (Zero Trust / Tunnel).
- `kubectl` configured to talk to the cluster.

## Secret Management

> *CRITICAL*: We do NOT store credentials in this repository. Before deploying the network layer, you must manually create the `tunnel-credentials` secret on the server.

1. Obtain token. Get your tunnel token from the [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/) (Networks > Connectors).
2. Create secret. Run this directly on your server:
    ```bash
    kubectl create secret generic tunnel-credentials \
        --from-literal=token='<YOUR_TOKEN>' \
        --namespace=default
    ```


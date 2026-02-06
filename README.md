# My Homelab Infrastructure

This repository contains the core infrastructure for my personal homelab. I use it to manage everything from networking and namespaces to cluster-wide configurations. My goal is to automate as much as possible, so I can spend more time building cool things and less time managing infrastructure.

## Table of Contents

- [Repository Structure](#repository-structure)
  - [Technology Stack](#technology-stack)
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Secret Management](#secret-management)
- [Deployment Guide](#deployment-guide)
- [Operations & Troubleshooting](#operations--troubleshooting)

## Repository Structure

- `core/`: This directory contains the core infrastructure for my homelab.
  - `networking/`: Networking configurations.
    - `cloudflared/`: Cloudflare Tunnel deployment (Dashboard Mode).
  - `cicd/`: CI/CD configurations.
    - `jenkins/`: Jenkins Helm values and RBAC permissions.
  - `git/`: Source control configurations.
    - `gitea/`: Self-hosted Gitea manifests (Deployment, Service, PVC).
  - `observability/`: Monitoring and Visualization.
    - `headlamp/`: Headlamp dashboard Helm values and Admin RBAC.
- `scripts/`: Utility scripts for management.
- `README.md`: Documentation.

### Technology Stack

* **Platform:** Kubernetes (K3s) on Ubuntu Server
* **Networking:** Cloudflare Tunnel (Zero Trust)
* **CI/CD:** Jenkins (Kaniko & Blue Ocean)
* **Version Control:** Gitea
* **Observability:** Headlamp (Kubernetes Dashboard)
* **Storage:** Local Path Provisioner

## Overview

I utilize a **Namespace-Layered Architecture** to keep the cluster clean and isolated:
* **`networking`**: Contains the Cloudflare Tunnel.
* **`git`**: Contains Gitea.
* **`cicd`**: Contains Jenkins.
* **`observability`**: Contains Headlamp.
* **`personal`**: (Planned) For personal projects like Portfolio.

## Prerequisites

- **Kubernetes Cluster**: K3s recommended.
- **Helm**: Required for Jenkins and Headlamp.
- **Cloudflare Account**: For the tunnel.
- **kubectl**: Configured for your cluster.

## Secret Management

Secrets must be created in their specific namespaces.

### 1. Cloudflare Tunnel

Required for the Cloudflare Tunnel to connect to the Cloudflare network.

```bash
kubectl create namespace networking

kubectl create secret generic tunnel-credentials \
    --from-literal=token='<YOUR_CLOUDFLARE_TOKEN>' \
    --namespace=networking
```

### 2. GitHub Credentials

Required for Jenkins to push images to GHCR.

```bash
kubectl create namespace cicd

kubectl create secret generic github-credentials \
    --namespace cicd \
    --from-literal=username='<GITHUB_USERNAME>' \
    --from-literal=password='<GITHUB_TOKEN>'

kubectl label secret github-credentials "jenkins.io/credentials-type=usernamePassword" -n cicd

kubectl annotate secret github-credentials "jenkins.io/credentials-description=GitHub User & Token" "jenkins.io/credentials-model-username-key=username" "jenkins.io/credentials-model-password-key=password" -n cicd
```

### 3. Gitea Credentials

Required for Jenkins to scan private Gitea repos.

```bash
kubectl create secret generic gitea-credentials \
    --namespace cicd \
    --from-literal=token='<YOUR_GITEA_TOKEN>'

kubectl label secret gitea-credentials "jenkins.io/credentials-type=secretText" -n cicd

kubectl annotate secret gitea-credentials "jenkins.io/credentials-description=Gitea Personal Access Token" "jenkins.io/credentials-model-secret-text-key=token" -n cicd
```

### 4. GHCR Pull Secrets

Required in any namespace where you pull private images (e.g., cicd for `agents`, `personal` for apps).

```bash
kubectl create secret docker-registry ghcr-pull-secrets \
    --docker-server=ghcr.io \
    --docker-username=<GITHUB_USERNAME> \
    --docker-password=<GITHUB_TOKEN> \
    --docker-email=<YOUR_EMAIL> \
    -n personal
```

## Deployment Guide

1.  **Networking Layer**:

    Deploy the Cloudflare Tunnel to the `networking` namespace.
    
    ```bash
    kubectl apply -f core/networking/cloudflared/cloudflared.yaml
    ```

2. **Git Layer**:
    
    Deploy Gitea to the `git` namespace.

    ```bash
    kubectl create namespace git

    kubectl apply -f core/git/gitea/gitea.yaml
    ```

3.  **Observability Layer (Headlamp)**:

    Deploy Headlamp to `observability` using Helm.

    ```bash
    kubectl create namespace observability

    kubectl apply -f core/observability/headlamp/rbac.yaml

    helm repo add headlamp [https://kubernetes-sigs.github.io/headlamp/](https://kubernetes-sigs.github.io/headlamp/)

    helm upgrade --install headlamp headlamp/headlamp \
        --namespace observability \
        --create-namespace \
        -f core/observability/headlamp/values.yaml
    ```

4.  **Deploy CI/CD (Jenkins)**:

    Deploy Jenkins to the `cicd` namespace.

    ```bash
    kubectl apply -f core/cicd/jenkins/rbac.yaml

    helm upgrade --install jenkins jenkins/jenkins \
        --namespace cicd \
        --create-namespace \
        -f core/cicd/jenkins/values.yaml
    ```

## Operations & Troubleshooting

### Accessing Headlamp Dashboard

Headlamp is secured via a Service Account Token.

1. Generate the Login Token:

    ```
    kubectl create token headlamp-admin -n observability --duration=2400h
    ```

2. Navigate to the dashboard URL and paste the token.

### SSH Access to Gitea

Gitea SSH is exposed on port 2222. You must configure your local SSH config or specific remote URLs to use this port.

* Hostname: `ssh-gitea.ahmadfaisal.space` (or your configured SSH domain)
* Port: `2222`

### Fixing a Stuck Jenkins Pod

If Jenkins gets stuck in a `Completed (0/2)` state after a server reboot, you can fix it by deleting the pod. Kubernetes will automatically create a new one.

```bash
kubectl delete pod jenkins-0 -n cicd
```

### Graceful Shutdown

To avoid corrupting the Jenkins or Gitea database, you should always shut down the server gracefully.

```bash
sudo shutdown now
```

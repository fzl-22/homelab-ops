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
  - `network/`: This directory contains the networking configuration for my homelab.
    - `tunnel.yaml`: This file defines the Cloudflare Tunnel that I use to expose my services to the internet.
  - `cicd/`: This directory contains the Jenkins configuration.
    - `jenkins-rbac.yaml`: Defines RBAC permissions for Jenkins to deploy to the cluster.
    - `jenkins-value.yaml`: Contains Helm chart overrides for Jenkins (plugins, resources, proxy settings).
- `scripts/`: This directory contains a collection of utility scripts that I use to manage my homelab.
- `README.md`: This file contains the documentation for my homelab.

### Technology Stack

## Overview

- **Cluster**: I'm running a lightweight K3s cluster on an old ASUS laptop running Ubuntu Server. I chose K3s because it's easy to install and manage, and it's perfect for a small homelab like mine.
- **Ingress**: I use a Cloudflare Tunnel to expose my services to the internet. This means I don't have to open any ports on my router, which is great for security. The tunnel is configured to point to my K3s cluster, so I can easily expose any service to the internet just by creating a new Ingress resource.
- **CI/CD**: I'm using Jenkins to build and deploy my applications. I've configured it to use Kaniko to build Docker images inside the cluster and store them into GHCR.
- **Deployments**: I'm using `kubectl` to deploy my applications. I've also set up RBAC to ensure that Jenkins only has the permissions it needs to deploy applications.

## Prerequisites

- **Kubernetes Cluster**: You'll need a Kubernetes cluster to run the applications. I'm using K3s, but any Kubernetes distribution should work.
- **Helm**: I'm using Helm to manage the Jenkins deployment. You'll need to have Helm installed and configured to work with your cluster.
- **Cloudflare Account**: I'm using a Cloudflare Tunnel to expose my services to the internet. You'll need a Cloudflare account to create and manage the tunnel.
- **kubectl**: You'll need `kubectl` to interact with your cluster. Make sure it's configured to talk to your cluster.

## Secret Management

I don't store any credentials in this repository. You'll need to create the following secrets manually before you can deploy the corresponding manifests.

### 1. Cloudflare Tunnel

This secret is required for the Cloudflare Tunnel to connect to the Cloudflare network.

```bash
kubectl create secret generic tunnel-credentials \
    --from-literal=token='<YOUR_CLOUDFLARE_TOKEN>' \
    --namespace=default
```

### 2. GitHub Credentials

This secret is required for Jenkins to authenticate with GitHub and push images to GHCR.

```bash
# 1. Create the secret
kubectl create secret generic github-credentials \
    --namespace jenkins \
    --from-literal=username='<GITHUB_USERNAME>' \
    --from-literal=password='<GITHUB_TOKEN>'

# 2. Label the secret for Jenkins
kubectl label secret github-credentials "jenkins.io/credentials-type=usernamePassword" -n jenkins
kubectl annotate secret github-credentials "jenkins.io/credentials-description=GitHub User & Token" "jenkins.io/credentials-model-username-key=username" "jenkins.io/credentials-model-password-key=password" -n jenkins
```

### 3. GHCR Pull Secrets

This secret is required for the cluster to pull private images from GHCR.

```bash
kubectl create secret docker-registry ghcr-pull-secrets \
    --docker-server=ghcr.io \
    --docker-username=<GITHUB_USERNAME> \
    --docker-password=<GITHUB_TOKEN> \
    --docker-email=<YOUR_EMAIL> \
    -n default
```

## Deployment Guide

1.  **Deploy the Cloudflare Tunnel**:

    ```bash
    kubectl apply -f core/network/tunnel.yaml
    ```

2.  **Set up RBAC for Jenkins**:

    ```bash
    kubectl apply -f core/cicd/jenkins-rbac.yaml
    ```

3.  **Install or update Jenkins**:

    ```bash
    helm upgrade --install jenkins jenkins/jenkins \
        --namespace jenkins \
        --create-namespace \
        -f core/cicd/jenkins-values.yaml
    ```

## Operations & Troubleshooting

### Fixing a Stuck Jenkins Pod

If Jenkins gets stuck in a `Completed (0/2)` state after a server reboot, you can fix it by deleting the pod. Kubernetes will automatically create a new one.

```bash
kubectl delete pod jenkins-0 -n jenkins
```

### Graceful Shutdown

To avoid corrupting the Jenkins database, you should always shut down the server gracefully.

```bash
sudo shutdown now
```

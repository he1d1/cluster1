# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a GitOps repository for a 3-node Raspberry Pi Kubernetes cluster running Talos OS. All infrastructure is managed declaratively through Flux CD.

## Key Technologies

- **Talos OS** (v1.11.5): Immutable Kubernetes-focused operating system
- **Talhelper**: Generates Talos machine configs from talconfig.yaml
- **Kubernetes** (v1.34.1): Container orchestration
- **Flux CD**: GitOps continuous delivery - reconciles cluster state from this repo
- **SOPS + Age**: Secret encryption in git
- **Longhorn**: Distributed block storage (replica-2)
- **Traefik**: Ingress controller with Gateway API

## Directory Structure

```
flux/
├── flux-system/     # Flux controller configuration
└── apps/            # Application deployments by namespace
    ├── cert-manager/    # TLS certificate automation (Let's Encrypt + Cloudflare)
    ├── longhorn-system/ # Distributed storage
    ├── media/           # Jellyfin, Radarr, Sonarr, Prowlarr, NZBGet
    ├── metallb/         # Bare-metal load balancer
    ├── monitoring/      # Prometheus, Grafana, Loki stack
    ├── promtail/        # Log aggregation (privileged namespace)
    └── traefik/         # Ingress controller

talos/
├── talconfig.yaml       # Talhelper cluster definition (nodes, network, extensions)
├── talsecret.sops.yaml  # Encrypted Talos secrets
└── clusterconfig/       # Generated per-node machine configs (via talhelper genconfig)
```

## Working with Secrets

Secrets are encrypted using SOPS with Age encryption. The `.sops.yaml` defines encryption rules:
- `flux/` path: encrypts `data` and `stringData` fields
- `talos/` path: full file encryption

To edit encrypted secrets:
```bash
sops flux/apps/<namespace>/secrets.sops.yaml
```

## Application Deployment Pattern

Each app namespace follows this structure:
```
app-name/
├── namespace.yaml      # Namespace definition
├── kustomization.yaml  # Kustomize configuration
├── helmrelease.yaml    # Helm chart deployment
├── helmrepository.yaml # Helm repo source
├── networkpolicy.yaml  # Network isolation
└── secrets.sops.yaml   # Encrypted secrets (if needed)
```

## Cluster Details

- **Nodes**: pi1 (192.168.1.21), pi2 (.22), pi3 (.23) - all control plane
- **Endpoint**: https://kube.cluster1.internal.heidi.codes:6443
- **Pod Network**: 10.244.0.0/16
- **LoadBalancer Range**: 192.168.1.100-192.168.1.110
- **Architecture**: ARM64

## Resource Constraints

This runs on Raspberry Pi hardware. Keep resource requests/limits conservative:
- Small apps: 25m CPU, 128Mi-256Mi memory
- Medium apps (Grafana, Loki): 128Mi request, 256Mi-512Mi limit
- Large apps (Prometheus, Jellyfin): 512Mi request, 2Gi limit

## Networking

- **HTTPRoute** resources define ingress routing (Gateway API, not legacy Ingress)
- **NetworkPolicy** resources control pod-to-pod communication
- TLS termination via Traefik with Let's Encrypt certificates
- **App hostnames**: `<app>.apps.cluster1.internal.heidi.codes` (use these for inter-app communication, not k8s service DNS)

## Flux Reconciliation

Changes pushed to this repo are automatically applied by Flux. To force reconciliation:
```bash
flux reconcile kustomization flux-system --with-source
```

Check status:
```bash
flux get all
kubectl get kustomizations -A
kubectl get helmreleases -A
```

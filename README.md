# Staer Robot Fleet - GitOps Repository

This repository contains the GitOps configuration for Staer robot deployments using Flux CD.

## Structure

Following the gitops2 pattern with three layers:

```
gitops/
├── clusters/stable/              # Entry point - Flux watches this
├── collections/stable/robot-fleet/  # Groups components together
└── components/apps/test-app/     # Individual component
    ├── base/                     # Base Kubernetes manifests
    ├── overlays/stable/          # Environment-specific config
    └── stable/                   # Flux CR wrapper
```

## Installation

The install script bootstraps a K3s cluster with Flux CD watching this repository:

```bash
curl -fsSL https://raw.githubusercontent.com/peterneubauer/staer-robot-fleet/main/scripts/install.sh | bash
```

## Verification

After installation, Flux will:
1. Watch the GitRepository at `https://github.com/peterneubauer/staer-robot-fleet`
2. Apply manifests from `./gitops/clusters/stable`
3. Create the `staer-test` namespace

Check Flux status:
```bash
kubectl get gitrepositories,kustomizations -n flux-system
kubectl get namespaces staer-test
```

## Architecture

- **K3s**: Lightweight Kubernetes for ARM/edge devices
- **Flux CD**: GitOps continuous delivery
- **Three-layer pattern**: Cluster → Collection → Component

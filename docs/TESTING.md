# Testing Flux GitOps on AGX Device

This document provides commands to verify that Flux is working correctly on your AGX device.

## Prerequisites

SSH access to the AGX device:
```bash
ssh user@100.100.168.87
# Password: admin
```

## Basic Flux Status Checks

### 1. Check Flux Controllers
Verify all Flux controllers are running:
```bash
sudo k3s kubectl get pods -n flux-system
```

Expected output: All pods should be `Running` with `READY 1/1`

### 2. Check GitRepository Status
Verify Flux is connected to the GitHub repository:
```bash
sudo k3s kubectl get gitrepositories -n flux-system
```

Expected output:
```
NAME                URL                                                  AGE   READY   STATUS
staer-robot-fleet   https://github.com/peterneubauer/staer-robot-fleet   ...   True    stored artifact for revision 'main@sha1:...'
```

### 3. Check Kustomizations
Verify Flux is applying manifests:
```bash
sudo k3s kubectl get kustomizations -n flux-system
```

Expected output:
```
NAME       AGE   READY   STATUS
cluster    ...   True    Applied revision: main@sha1:...
test-app   ...   True    Applied revision: main@sha1:...
```

### 4. Verify Test Namespace Created
Check that Flux created the test namespace:
```bash
sudo k3s kubectl get namespace staer-test
```

Expected output:
```
NAME         STATUS   AGE
staer-test   Active   ...
```

Detailed view with labels:
```bash
sudo k3s kubectl get namespace staer-test -o yaml | grep -A 5 labels
```

Should show:
- `purpose: flux-verification` (our custom label)
- `kustomize.toolkit.fluxcd.io/name: test-app` (Flux tracking)

## Testing GitOps Updates

### Test 1: Add a ConfigMap via Git

From your local machine:

```bash
# Navigate to the repo
cd ~/src/staer/staer-robot-fleet

# Create a test ConfigMap
cat > gitops/components/apps/test-app/base/configmap.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-config
  namespace: staer-test
data:
  message: "Hello from GitOps!"
  timestamp: "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
EOF

# Add it to the kustomization
cat gitops/components/apps/test-app/base/kustomization.yaml
# Add configmap.yaml to resources list if not already there
```

Edit the kustomization to include the configmap:
```bash
cat > gitops/components/apps/test-app/base/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - configmap.yaml
EOF
```

Commit and push:
```bash
git add gitops/components/apps/test-app/base/
git commit -m "test: add ConfigMap to test GitOps updates"
git push
```

On the AGX device, wait ~1 minute then check:
```bash
# Check if Flux detected the update
sudo k3s kubectl get gitrepositories -n flux-system -o jsonpath='{.items[0].status.artifact.revision}'

# Check if ConfigMap was created
sudo k3s kubectl get configmap -n staer-test

# View the ConfigMap content
sudo k3s kubectl get configmap test-config -n staer-test -o yaml
```

### Test 2: Force Immediate Reconciliation

If you don't want to wait for Flux's 1-minute interval:

```bash
# Force GitRepository sync
sudo k3s kubectl annotate gitrepository staer-robot-fleet -n flux-system \
  reconcile.fluxcd.io/requestedAt="$(date +%Y-%m-%dT%H:%M:%SZ)" --overwrite

# Force Kustomization sync
sudo k3s kubectl annotate kustomization test-app -n flux-system \
  reconcile.fluxcd.io/requestedAt="$(date +%Y-%m-%dT%H:%M:%SZ)" --overwrite
```

Then immediately check status:
```bash
sudo k3s kubectl get kustomizations -n flux-system
```

### Test 3: Verify GitOps Deletion

From your local machine, remove a resource:

```bash
cd ~/src/staer/staer-robot-fleet

# Remove the ConfigMap file
rm gitops/components/apps/test-app/base/configmap.yaml

# Update kustomization
cat > gitops/components/apps/test-app/base/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
EOF

git add -A
git commit -m "test: remove ConfigMap to test GitOps pruning"
git push
```

On the AGX device (after ~1 minute):
```bash
# ConfigMap should be deleted (because prune: true in Flux CR)
sudo k3s kubectl get configmap -n staer-test
# Should show: No resources found
```

## Troubleshooting Commands

### View Flux Logs

If something isn't working, check Flux controller logs:

```bash
# Source controller (handles GitRepository)
sudo k3s kubectl logs -n flux-system deployment/source-controller --tail=50

# Kustomize controller (applies manifests)
sudo k3s kubectl logs -n flux-system deployment/kustomize-controller --tail=50
```

### Check for Errors

```bash
# Check GitRepository for errors
sudo k3s kubectl describe gitrepository staer-robot-fleet -n flux-system

# Check Kustomization for errors
sudo k3s kubectl describe kustomization test-app -n flux-system
```

### View All Flux-Managed Resources

See everything Flux has created:
```bash
sudo k3s kubectl get all,configmaps,secrets -n staer-test
```

## Quick Verification Script

Save this as `test-flux.sh` on the device:

```bash
#!/bin/bash
echo "=== Flux Controllers ==="
sudo k3s kubectl get pods -n flux-system

echo -e "\n=== GitRepository Status ==="
sudo k3s kubectl get gitrepositories -n flux-system

echo -e "\n=== Kustomizations ==="
sudo k3s kubectl get kustomizations -n flux-system

echo -e "\n=== Resources in staer-test namespace ==="
sudo k3s kubectl get all,configmaps -n staer-test

echo -e "\n=== Recent Flux Events ==="
sudo k3s kubectl get events -n flux-system --sort-by='.lastTimestamp' | tail -10
```

Make it executable and run:
```bash
chmod +x test-flux.sh
./test-flux.sh
```

## Expected Behavior

1. **GitRepository** syncs every 1 minute (configurable in the Flux CR)
2. **Kustomization** applies changes within seconds of GitRepository update
3. **Prune: true** means deleted resources are automatically removed from cluster
4. **Changes are automatic** - no manual kubectl apply needed!

## Current Structure

```
Flux watches: https://github.com/peterneubauer/staer-robot-fleet
Entry point: ./gitops/clusters/stable
Current resources:
  - Namespace: staer-test (with label: purpose=flux-verification)
```

## Next Steps

Once basic testing is working:
1. Add actual application deployments (staer-client, etc.)
2. Add infrastructure components (secrets, storage, etc.)
3. Set up image automation for automatic updates
4. Add monitoring and alerting

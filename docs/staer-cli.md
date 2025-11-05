# Staer CLI - Robot Cluster Management

The `staer` CLI is a minimal command-line tool installed on robot devices for managing the local K3s cluster and Staer robot components.

## Installation

The CLI is automatically installed to `/usr/local/bin/staer` during device provisioning via the installation script.

## Prerequisites

- K3s cluster installed and running
- Flux CD bootstrapped with GitOps repository
- Staer namespace and components deployed

## Commands

### `staer up`
Start the K3s cluster service.

```bash
staer up
```

**Note**: If the cluster is already running, this command will notify you without making changes.

### `staer down`
Stop the K3s cluster service.

```bash
staer down
```

**Warning**: This stops the entire Kubernetes cluster including all running pods.

### `staer status` (default)
Display comprehensive cluster status including:
- Cluster connectivity and version
- All pods across all namespaces
- Flux CD GitRepository and Kustomization status
- Staer namespace pods

```bash
staer status
# or simply
staer
```

**Example output**:
```
=== Cluster Info ===
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

=== All Pods ===
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
flux-system   helm-controller-846b49d5ff-mprn2          1/1     Running   0          55m
flux-system   image-automation-controller-6ccb7bb695    1/1     Running   0          55m
...

=== Flux Status ===
NAME                URL                                                  AGE   READY   STATUS
staer-robot-fleet   https://github.com/peterneubauer/staer-robot-fleet   55m   True    stored artifact...

=== Staer Namespace ===
No staer pods found
```

### `staer logs`
Follow logs from the staer-client pod in real-time.

```bash
staer logs
```

**Requirements**:
- Staer-client pod must be running
- Pod must have label `app=staer-client`

### `staer shell`
Open an interactive bash shell inside the staer-client pod.

```bash
staer shell
```

**Requirements**:
- Staer-client pod must be running
- Pod must have label `app=staer-client`

**Example session**:
```bash
staer shell
# Inside pod:
ros2 topic list
ros2 node list
exit
```

### `staer restart`
Restart the staer-client deployment with zero-downtime rollout.

```bash
staer restart
```

This triggers a Kubernetes rolling restart and waits for the deployment to become ready.

### `staer update`
Trigger Flux CD to reconcile immediately, pulling latest changes from the GitOps repository.

```bash
staer update
```

This is useful when you've pushed changes to the git repository and want them applied immediately rather than waiting for the automatic sync interval.

### `staer help`
Display help message with all available commands and examples.

```bash
staer help
# or
staer --help
staer -h
```

## Usage Examples

### Check if everything is running
```bash
staer status
```

### Start development session
```bash
# Start cluster if stopped
staer up

# Check status
staer status

# View logs
staer logs
```

### Apply GitOps changes
```bash
# After pushing changes to git repository
staer update

# Wait a moment, then check status
sleep 5
staer status
```

### Debug a pod
```bash
# Get interactive shell
staer shell

# Inside pod, run diagnostics
env | grep STAER
ps aux
ros2 doctor --report

# Exit
exit
```

### Restart after configuration change
```bash
# Update GitOps repo with new configuration
staer update

# Force restart of staer-client
staer restart

# Verify new configuration loaded
staer logs
```

## Permissions

Most commands require `sudo` privileges because they interact with the K3s cluster through `k3s kubectl`, which requires root access.

```bash
sudo staer status
sudo staer restart
sudo staer logs
```

## Technical Details

### Architecture
- **Cluster**: K3s (lightweight Kubernetes)
- **GitOps**: Flux CD syncing from GitHub
- **Namespace**: `staer` for robot components
- **Pod label**: `app=staer-client` for pod selection

### kubectl Integration
The CLI uses `k3s kubectl` under the hood. All kubernetes operations are performed through:
```bash
KUBECTL="k3s kubectl"
sudo $KUBECTL <command>
```

### Service Management
K3s runs as a systemd service, managed through:
```bash
sudo systemctl start k3s    # Start cluster
sudo systemctl stop k3s     # Stop cluster
sudo systemctl status k3s   # Check service status
```

## Troubleshooting

### "Cluster not running"
The K3s service is not active. Start it with:
```bash
staer up
```

### "No staer-client pod found"
The staer-client deployment is not yet running. Check Flux sync status:
```bash
staer status
# Look for Flux kustomization status
```

If Flux shows errors, check the GitOps repository for configuration issues.

### Flux not reconciling
Manually trigger reconciliation:
```bash
staer update
```

Wait 10-15 seconds and check again:
```bash
staer status
```

### kubectl permission denied
All cluster operations require sudo:
```bash
sudo staer <command>
```

## Configuration

### Device Information
Device-specific configuration is stored in `/etc/staer/device-info.json`:
```json
{
  "device_uuid": "32029AB7-DDE3-40A5-9D83-E1CCC2A0F98F",
  "os": "ubuntu",
  "os_version": "22.04",
  "architecture": "aarch64",
  "flux_repo_url": "https://github.com/peterneubauer/staer-robot-fleet",
  "flux_branch": "main",
  "flux_path": "./gitops/clusters/stable"
}
```

### Cluster Configuration
- **Cluster name**: `staer-robot`
- **Namespace**: `staer`
- **Flux namespace**: `flux-system`
- **GitRepository name**: `staer-robot-fleet`
- **Kustomization name**: `cluster`

## Related Documentation

- [Installation Guide](../README.md) - Initial device setup
- [GitOps Structure](./gitops-structure.md) - Repository layout and components
- [Flux CD Documentation](https://fluxcd.io/docs/) - Upstream Flux documentation
- [K3s Documentation](https://docs.k3s.io/) - K3s cluster management

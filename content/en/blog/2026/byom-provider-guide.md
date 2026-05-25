---
date: 2026-05-25
title: "BYOM Provider: Bring Your Own CVMs to run Confidential Containers"
linkTitle: "BYOM Provider for Confidential Containers"
description: >
  Learn how to use the BYOM (Bring Your Own Machine) provider to leverage pre-created CVMs to run Kubernetes pods
author: "[Amulya Meka](https://github.com/Amulyam24)"
---

## Introduction

The BYOM (Bring Your Own Machine) provider is a unique Cloud API Adaptor (CAA) provider that enables you to use pre-created CVMs as peer pods. Unlike other CAA providers (AWS, Azure, GCP, etc.) that dynamically provision CVMs, BYOM manages a pool of pre-existing CVMs, allocating them to workloads and returning them to the pool when done.

This approach is valuable for on-premises deployments, environments with longer VM provisioning times, or scenarios requiring complete control over VM lifecycle. This blog explores BYOM's architecture, features, and practical usage, including an example using Docker containers for local development.

## What Makes BYOM Different?

The BYOM provider stands out in several key ways:

- **No VM Lifecycle Management**: VMs are pre-created and managed externally; BYOM only allocates and deallocates them for pods
- **Infrastructure Agnostic**: Works with any infrastructure: cloud, on-premises, bare metal, or even containers
- **SSH/SFTP Communication**: Unlike other providers that pass user-data during VM creation via cloud-init, BYOM delivers configuration to pre-existing VMs by writing directly to `/media/cidata/user-data` via SFTP after allocation
- **Distributed State Management**: Coordinates VM allocation across multiple CAA instances using Kubernetes ConfigMaps

### Important Considerations

The BYOM model has some characteristics to note:

- **Pre-existing Infrastructure**: BYOM assumes pre-created CVMs are in a trusted state
- **CVM Reuse**: CVMs are reused across different workloads though BYOM triggers a reboot after each use to ensure clean state


## Architecture Overview

### Core Components

The BYOM provider consists of several key components:

1. **VM Pool Manager**: Manages the global pool of available VM IP addresses
2. **ConfigMap-based State**: Stores allocation state persistently
3. **SSH/SFTP Client**: Delivers pod configuration (user-data) to VMs securely
4. **Optimistic Locking**: Handles concurrent allocation requests from multiple CAA instances
5. **State Recovery**: Restores allocation state after CAA restarts

### How It Works

```mermaid
graph TB
    User["👤 User<br/>(kubectl create pod)"]
    
    subgraph Kubernetes["Kubernetes Cluster"]
        subgraph CAA["Cloud API Adaptor (CAA)"]
            BYOM["BYOM Provider"]
            Pool["VM Pool Manager<br/>(Hash-based Selection)"]
            CM["🗂️ ConfigMap<br/>byom-ip-pool-state<br/>(Persistent State)"]
        end
        PodRequest["☸️ Pod Request<br/>(Triggers VM Allocation)"]
    end
    
    subgraph Infrastructure["Pre-existing Infrastructure"]
        VM1["🖥️ VM: 192.168.1.10<br/>(Available)"]
        VM2["🖥️ VM: 192.168.1.11<br/>(Becomes Peer Pod)"]
        VM3["🖥️ VM: 192.168.1.12<br/>(Available)"]
    end
    
    User -->|"Create Pod"| PodRequest
    PodRequest -->|"① Request Instance"| BYOM
    BYOM -->|"② Allocate IP"| Pool
    Pool <-->|"③ Read/Update State<br/>(Optimistic Locking)"| CM
    Pool -->|"④ Return IP"| BYOM
    BYOM -->|"⑤ SSH/SFTP<br/>Send user-data to<br/>/media/cidata/user-data"| VM2
    VM2 -.->|"⑥ VM Becomes Peer Pod<br/>(Runs Workload)"| PodRequest
    
    BYOM -->|"⑦ On Delete:<br/>Send reboot trigger to<br/>/media/cidata/reboot"| VM2
    VM2 -.->|"⑧ Reboot & Return to Pool"| Pool
    
    style User fill:#e8f4f8,stroke:#0288d1,stroke-width:3px
    style PodRequest fill:#e1f5ff,stroke:#0066cc,stroke-width:3px
    style BYOM fill:#ffe1f5,stroke:#cc0088,stroke-width:2px
    style Pool fill:#e1ffe1,stroke:#00cc44,stroke-width:2px
    style CM fill:#f5e1ff,stroke:#8800cc,stroke-width:2px
    style VM1 fill:#d4edda,stroke:#28a745,stroke-width:2px
    style VM2 fill:#fff3cd,stroke:#ffc107,stroke-width:2px
    style VM3 fill:#d4edda,stroke:#28a745,stroke-width:2px
    style Kubernetes fill:#f0f8ff,stroke:#4682b4,stroke-width:2px
    style CAA fill:#fff9e6,stroke:#ff9800,stroke-width:2px
    style Infrastructure fill:#ffe6e6,stroke:#dc3545,stroke-width:2px
```

**Pod Creation Flow:**

1. **Request Instance**: Pod request triggers the BYOM provider to allocate a VM
2. **Allocate IP**: BYOM provider requests an IP from the VM Pool Manager
3. **Read/Update State**: VM Pool Manager reads ConfigMap, selects IP using hash-based algorithm, and updates state with optimistic locking
4. **Return IP**: VM Pool Manager returns the allocated IP to BYOM provider
5. **Send Configuration**: BYOM provider connects via SSH/SFTP and writes pod configuration (user-data) to `/media/cidata/user-data` on the VM
6. **VM Becomes Peer Pod**: The VM processes the user-data and becomes the peer pod, running the confidential workload

**Pod Deletion Flow:**

7. **Send Reboot Trigger**: When the peer pod is deleted, BYOM provider sends a reboot trigger file to `/media/cidata/reboot` via SFTP
8. **Return to Pool**: The VM reboots, cleans up its state, and the IP is returned to the available pool in the ConfigMap for reuse

## Key Features

### 1. Flexible IP Pool Configuration

BYOM supports multiple ways to specify your VM pool:

```yaml
# Individual IPs
VM_POOL_IPS: "192.168.1.10,192.168.1.11,192.168.1.12"

# IP ranges (up to 100 IPs per range by default)
VM_POOL_IPS: "192.168.1.10-192.168.1.50"

# Mixed format
VM_POOL_IPS: "192.168.1.10,192.168.1.20-192.168.1.30,192.168.1.100"
```

The provider automatically validates all IP addresses, expands ranges, removes duplicates, and enforces the maximum range limit (configurable via `MAX_RANGE_IPS`).

### 2. SSH Security Modes

BYOM provides two SSH host key verification modes:

**Stateless TOFU (Trust On First Use)** - Default mode:
- Accepts any SSH host key during connection
- Logs the key fingerprint for security monitoring
- Suitable for development and testing

**Host Key Allowlist** - Recommended for production:
- Only accepts pre-approved SSH host keys
- Rejects connections from VMs not in the allowlist
- Enable by setting `SSH_HOST_KEY_ALLOWLIST_DIR`

To extract and configure host keys:

```bash
# Extract host keys from a VM
ssh-keyscan -t rsa 192.168.1.10 | sed 's/^[^ ]* //' > vm1_rsa.pub
ssh-keyscan -t ecdsa 192.168.1.10 | sed 's/^[^ ]* //' > vm1_ecdsa.pub
ssh-keyscan -t ed25519 192.168.1.10 | sed 's/^[^ ]* //' > vm1_ed25519.pub

# Verify fingerprints
ssh-keygen -lf vm1_ecdsa.pub
```

### 3. Distributed IP Allocation

The BYOM provider uses a sophisticated allocation strategy to minimize conflicts when multiple CAA instances run simultaneously:

**Hash-based Selection**: Each allocation ID (derived from pod name and sandbox ID) is hashed to select an IP from the available pool. This ensures different pods typically select different IPs, reducing contention between CAA instances.

**Optimistic Locking**: Uses Kubernetes ConfigMap ResourceVersion for conflict detection:
1. Read current state from ConfigMap
2. Select and allocate an IP
3. Update ConfigMap with new state
4. If ResourceVersion conflict occurs, retry with fresh state

**Viewing Allocation State**:

You can inspect the current state of IP allocations at any time:

```bash
kubectl get configmap byom-ip-pool-state -n confidential-containers-system -o jsonpath='{.data.allocation-state}' | jq .
```

Example output showing allocated and available IPs:

```json
{
  "allocatedIPs": {
    "nginx-pod-abc123": {
      "allocationID": "nginx-pod-abc123",
      "ip": "192.168.1.10",
      "nodeName": "worker-1",
      "podName": "nginx-pod",
      "allocatedAt": "2026-05-25T10:30:00Z"
    }
  },
  "availableIPs": [
    "192.168.1.11",
    "192.168.1.12"
  ],
  "lastUpdated": "2026-05-25T10:30:00Z",
  "version": 42
}
```

This shows which IPs are currently allocated to which pods, and which IPs are available for new allocations.

### 4. Automatic State Recovery

When a CAA instance restarts, the BYOM provider automatically detects the current node name, loads allocation state from the ConfigMap, and reconciles any inconsistencies to ensure the VM pool reflects the configured IPs.

## Getting Started

### Prerequisites

1. **SSH Key Pair**: Generate keys for VM authentication

```bash
ssh-keygen -f ./id_rsa -N ""
```

This creates `id_rsa` (private key) and `id_rsa.pub` (public key).

2. **Pre-created VMs**: You need VMs ready with one of these approaches:
   - Custom PodVM image with embedded SSH keys
   - Existing VMs configured with the setup script

### Option 1: Build Custom PodVM Image

Build an SFTP-enabled PodVM image with embedded SSH keys:

```bash
# Copy your public key
cp id_rsa.pub ./resources/authorized_keys

# Build the image using mkosi
cd src/cloud-api-adaptor/podvm-mkosi
make sftp

# The image will be in ./build/podvm-fedora-amd64.qcow2
```

For Ubuntu images:

```bash
PODVM_DISTRO=ubuntu make sftp
```

Create VMs using the generated image. For example, with LibVirt:

```bash
# Copy image to libvirt directory
cp ./build/podvm-fedora-amd64.qcow2 /var/lib/libvirt/images

# Create VM
virt-install \
   --name podvm-test \
   --boot uefi \
   --memory 2048 \
   --vcpus 1 \
   --import \
   --os-variant fedora41 \
   --network default \
   --disk /var/lib/libvirt/images/podvm-fedora-amd64.qcow2 \
   --noautoconsole

# Get VM IP
virsh domifaddr podvm-test
```

### Option 2: Configure Existing VMs

Use the automated setup script to configure existing VMs:

```bash
# Prerequisites: yq, Golang, Docker

# Clone the repository
git clone https://github.com/confidential-containers/cloud-api-adaptor.git
cd cloud-api-adaptor/src/cloud-api-adaptor

# Run setup script with your public key
SSH_PUBLIC_KEY_PATH=/path/to/id_rsa.pub ./hack/setup-podvm-byom.sh
```

### Deployment with Helm

1. **Create secrets file** with your SSH keys:

```bash
cp src/cloud-api-adaptor/install/charts/peerpods/providers/byom-secrets.yaml.template \
   src/cloud-api-adaptor/install/charts/peerpods/providers/byom-secrets.yaml
```

Edit `byom-secrets.yaml`:

```yaml
providerSecrets:
  byom:
    id_rsa: |
      -----BEGIN OPENSSH PRIVATE KEY-----
      <paste your private key content here>
      -----END OPENSSH PRIVATE KEY-----
    id_rsa_pub: |
      ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC... user@host
```

{{% alert color="warning" %}}
Both `id_rsa` (private key) and `id_rsa.pub` (public key) are required for the BYOM provider to function correctly.
{{% /alert %}}

2. **Configure VM pool IPs** in `providers/byom.yaml`:

```yaml
providerConfigs:
  byom:
    VM_POOL_IPS: "192.168.1.10,192.168.1.11,192.168.1.12"
```

3. **Deploy CAA**:

```bash
export CLOUD_PROVIDER=byom
make deploy
```

## Configuration Reference

### Required Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `VM_POOL_IPS` | Comma-separated list of VM IPs or ranges | (required) |
| `SSH_USERNAME` | SSH username for VM access | `peerpod` |

### Optional Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `SSH_TIMEOUT` | SSH connection timeout (seconds) | `30` |
| `SSH_HOST_KEY_ALLOWLIST_DIR` | Directory with allowed host keys | `""` (TOFU mode) |
| `POOL_NAMESPACE` | Namespace for ConfigMap storage | Auto-detected |
| `POOL_CONFIGMAP_NAME` | ConfigMap name for state | `byom-ip-pool-state` |
| `MAX_RANGE_IPS` | Maximum IPs per range | `100` |
| `SSH_PRIV_KEY_PATH` | SSH private key file path | `/root/.ssh/id_rsa` |
| `SSH_PUB_KEY_PATH` | SSH public key file path | `/root/.ssh/id_rsa.pub` |

{{% alert color="info" %}}
The default namespace for ConfigMap storage is `confidential-containers-system`. If not explicitly set, BYOM auto-detects the namespace from the running pod's service account.
{{% /alert %}}

## Example: Using Docker Containers as VMs

An interesting use case for BYOM is using Docker containers as "VMs" for local development and testing with KinD (Kubernetes in Docker) clusters.

### Building the BYOM Docker Image

The BYOM provider includes a special Docker image that runs a containerized PodVM with SSH/SFTP support:

```bash
cd src/cloud-api-adaptor/podvm-mkosi

# Build the base Docker PodVM image
make container

# Build the BYOM variant with SSH support
make byom-e2e-container
```

This creates a container image based on `Dockerfile.podvm_byom_docker_provider.ubuntu` that includes:
- SSH server with SFTP support
- The `peerpod` user for SFTP access
- Systemd services for configuration processing
- Reboot watcher for cleanup

### Setting Up with Kind

1. **Create a Kind cluster**:

```bash
kind create cluster --config src/cloud-api-adaptor/byom/e2e/kind-config.yaml
```

The configuration mounts Docker socket and volumes needed for the containers to function as PodVMs.

2. **Start Docker containers** to act as your VM pool:

```bash
# Start multiple containers
for i in {1..3}; do
  docker run -d \
    --name podvm-$i \
    --privileged \
    --network kind \
    quay.io/confidential-containers/podvm-byom-e2e-image:latest
done

# Get container IPs
for i in {1..3}; do
  docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' podvm-$i
done
```

3. **Configure BYOM** to use the container IPs in `providers/byom.yaml`:

```yaml
providerConfigs:
  byom:
    VM_POOL_IPS: "<container-ip-1>,<container-ip-2>,<container-ip-3>"
```

4. **Deploy CAA** with the BYOM provider as described in the [Deployment with Helm](#deployment-with-helm) section.

Now your peer pods will run inside these Docker containers, which behave like VMs from the BYOM provider's perspective. This setup is excellent for:
- Local development and testing
- Learning and experimentation
- Debugging CAA and peer pod functionality

### Common Issues

**SSH Connection Failures**:
- Verify SSH keys are correctly configured in secrets
- Check VM network connectivity
- Ensure SSH service is running on VMs
- Review host key verification settings

**IP Allocation Conflicts**:
- Check for duplicate IPs in `VM_POOL_IPS`
- Verify ConfigMap is accessible
- Review CAA logs for retry attempts

**VM Not Ready**:
- Ensure VMs have the correct SSH keys
- Verify SFTP chroot directory exists (`/media/cidata`)
- Check VM logs for configuration errors

### Debugging Commands

```bash
# View CAA logs
kubectl logs -n confidential-containers-system daemonset/cloud-api-adaptor-daemonset

# Test SSH connectivity
ssh -i id_rsa peerpod@192.168.1.10

# Check SFTP access
sftp -i id_rsa peerpod@192.168.1.10

# View VM allocation history
kubectl get configmap byom-ip-pool-state -n confidential-containers-system -o jsonpath='{.data.allocation-state}' | jq .
```

## Use Cases

BYOM is particularly well-suited for:

1. **On-Premises Deployments**: Leverage existing bare-metal or VM infrastructure
2. **Hybrid Cloud**: Mix cloud and on-premises resources
3. **Custom Infra Environment**: Environments with special networking and storage requirements
4. **Development/Testing**: Use local VMs or containers for rapid iteration

## Security Considerations

1. **SSH Key Management**:
   - Store private keys securely in Kubernetes Secrets
   - Rotate keys regularly
   - Use separate keys for different environments

2. **Host Key Verification**:
   - Use allowlist mode in production
   - Regularly audit host key fingerprints
   - Monitor for unexpected key changes

## Conclusion

The BYOM provider offers a unique approach to peer pod management in Confidential Containers, enabling organizations to leverage existing infrastructure while maintaining the benefits of the CAA architecture. Its distributed state management, flexible configuration, and infrastructure-agnostic design make it suitable for a wide range of deployment scenarios.

## References

- [BYOM Provider Documentation](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/byom/README.md)
- [IP Allocation Design](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-providers/byom/ip_allocation.md)
- [Cloud API Adaptor Repository](https://github.com/confidential-containers/cloud-api-adaptor)

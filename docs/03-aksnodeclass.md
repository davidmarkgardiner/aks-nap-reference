# AKSNodeClass Configuration

AKSNodeClass defines Azure-specific settings for nodes. Each NodePool references an AKSNodeClass.

## Basic Structure

```yaml
apiVersion: karpenter.azure.com/v1beta1
kind: AKSNodeClass
metadata:
  name: default
spec:
  imageFamily: Ubuntu2204
  osDiskSizeGB: 128
  maxPods: 30
```

## Image Family

Controls the OS image:

```yaml
spec:
  # Options: Ubuntu2204 (default), AzureLinux
  imageFamily: Ubuntu2204
```

### FIPS Compliance

```yaml
spec:
  # Options: Disabled (default), FIPS
  fipsMode: FIPS
```

## OS Disk Configuration

### Disk Size

```yaml
spec:
  osDiskSizeGB: 128  # Default: 128, Min: 30
```

### Ephemeral OS Disk

NAP automatically uses ephemeral disks when:
- VM supports ephemeral OS disks
- Ephemeral capacity >= requested osDiskSizeGB
- VM has sufficient ephemeral storage

**Priority order:** NVMe > Cache > Resource disks

To ensure ephemeral disk usage:

```yaml
# In NodePool
requirements:
  - key: karpenter.azure.com/sku-storage-ephemeralos-maxsize
    operator: Gt
    values: ["128"]  # Require > 128GB ephemeral
```

## Network Configuration

### VNet Subnet

```yaml
spec:
  vnetSubnetID: "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}/subnets/{subnet}"
```

If not specified, uses the default from NAP installation.

## Pod Density

### Max Pods per Node

```yaml
spec:
  maxPods: 50  # Range: 10-250
```

**Defaults by network plugin:**
| Network Config | Default maxPods |
|----------------|-----------------|
| Azure CNI (NodeSubnet) | 30 |
| Azure CNI Overlay | 250 |
| None | 250 |
| Other | 110 |

## LocalDNS

Node-level DNS proxy for reduced latency:

```yaml
spec:
  LocalDNS:
    mode: Preferred  # Disabled, Preferred, Required
```

## Kubelet Configuration

### CPU Management

```yaml
spec:
  kubelet:
    cpuManagerPolicy: "static"  # "none" or "static"
    cpuCFSQuota: true
    cpuCFSQuotaPeriod: "100ms"
```

### Topology Manager

```yaml
spec:
  kubelet:
    # Options: none, restricted, best-effort, single-numa-node
    topologyManagerPolicy: "best-effort"
```

### Image Garbage Collection

```yaml
spec:
  kubelet:
    imageGCHighThresholdPercent: 85  # Trigger GC
    imageGCLowThresholdPercent: 80   # Target after GC
```

### Container Logs

```yaml
spec:
  kubelet:
    containerLogMaxSize: "50Mi"
    containerLogMaxFiles: 5
```

### Process Limits

```yaml
spec:
  kubelet:
    podPidsLimit: 4096  # -1 for unlimited
```

### Unsafe Sysctls

```yaml
spec:
  kubelet:
    allowedUnsafeSysctls:
      - "kernel.msg*"
      - "net.ipv4.route.min_pmtu"
```

## Resource Tags

Azure tags applied to all VMs:

```yaml
spec:
  tags:
    Environment: production
    Team: platform
    CostCenter: engineering
```

## Complete Example

```yaml
apiVersion: karpenter.azure.com/v1beta1
kind: AKSNodeClass
metadata:
  name: production
spec:
  # OS Configuration
  imageFamily: Ubuntu2204
  fipsMode: Disabled
  osDiskSizeGB: 256
  
  # Network
  vnetSubnetID: "/subscriptions/xxx/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/myVNet/subnets/nodeSubnet"
  
  # Pod Density
  maxPods: 110
  
  # DNS
  LocalDNS:
    mode: Preferred
  
  # Tags
  tags:
    Environment: production
    Team: platform
    ManagedBy: karpenter
  
  # Kubelet
  kubelet:
    cpuManagerPolicy: "static"
    cpuCFSQuota: true
    topologyManagerPolicy: "best-effort"
    imageGCHighThresholdPercent: 85
    imageGCLowThresholdPercent: 80
    containerLogMaxSize: "100Mi"
    containerLogMaxFiles: 5
    podPidsLimit: 4096
```

## Linking NodePool to AKSNodeClass

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: my-nodepool
spec:
  template:
    spec:
      nodeClassRef:
        group: karpenter.azure.com
        kind: AKSNodeClass
        name: production  # Reference to AKSNodeClass
```

Multiple NodePools can reference the same AKSNodeClass.

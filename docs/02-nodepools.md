# NodePool Configuration

NodePools define constraints on nodes Karpenter can create and pods that can run on them.

## Default NodePool

NAP creates a default NodePool automatically:

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
  template:
    spec:
      nodeClassRef:
        name: default
      expireAfter: Never
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: [amd64]
        - key: kubernetes.io/os
          operator: In
          values: [linux]
        - key: karpenter.sh/capacity-type
          operator: In
          values: [on-demand]
        - key: karpenter.azure.com/sku-family
          operator: In
          values: [D]
```

## SKU Selectors (Azure-specific)

### VM Family Selection

```yaml
requirements:
  - key: karpenter.azure.com/sku-family
    operator: In
    values:
      - D    # General purpose
      - F    # Compute optimized
      - E    # Memory optimized
      - L    # Storage optimized
      - N    # GPU-enabled
```

### Specific SKU Names

```yaml
requirements:
  - key: karpenter.azure.com/sku-name
    operator: In
    values:
      - Standard_D4s_v3
      - Standard_D8s_v3
      - Standard_F8s_v2
```

### SKU Version

```yaml
requirements:
  - key: karpenter.azure.com/sku-version
    operator: In
    values:
      - "3"  # v3 generation
      - "5"  # v5 generation
```

### CPU/Memory Requirements

```yaml
requirements:
  - key: karpenter.azure.com/sku-cpu
    operator: Gt
    values: ["4"]
  - key: karpenter.azure.com/sku-memory
    operator: Gt
    values: ["16384"]  # MiB
```

### GPU Selection

```yaml
requirements:
  - key: karpenter.azure.com/sku-gpu-name
    operator: In
    values: [A100, V100]
  - key: karpenter.azure.com/sku-gpu-manufacturer
    operator: In
    values: [nvidia]
  - key: karpenter.azure.com/sku-gpu-count
    operator: Gt
    values: ["1"]
```

### Storage Requirements

```yaml
requirements:
  - key: karpenter.azure.com/sku-networking-accelerated
    operator: In
    values: ["true"]
  - key: karpenter.azure.com/sku-storage-premium-capable
    operator: In
    values: ["true"]
  - key: karpenter.azure.com/sku-storage-ephemeralos-maxsize
    operator: Gt
    values: ["128"]  # GB
```

## Kubernetes Well-Known Labels

### Availability Zones

```yaml
requirements:
  - key: topology.kubernetes.io/zone
    operator: In
    values:
      - uksouth-1
      - uksouth-2
      - uksouth-3
```

### Architecture

```yaml
requirements:
  - key: kubernetes.io/arch
    operator: In
    values:
      - amd64
      - arm64
```

### Capacity Type (Spot/On-Demand)

```yaml
requirements:
  - key: karpenter.sh/capacity-type
    operator: In
    values:
      - spot
      - on-demand
```

**Note:** NAP prioritizes Spot when both are specified.

## Resource Limits

Prevent runaway scaling:

```yaml
spec:
  limits:
    cpu: "1000"       # Total vCPUs
    memory: 1000Gi    # Total memory
```

## Node Pool Weights

When multiple NodePools match, higher weight wins:

```yaml
spec:
  weight: 10  # Higher = more preferred
```

## Taints

### Regular Taints (pods must tolerate)

```yaml
spec:
  template:
    spec:
      taints:
        - key: dedicated
          value: gpu-workloads
          effect: NoSchedule
```

### Startup Taints (temporary, removed by DaemonSet)

```yaml
spec:
  template:
    spec:
      startupTaints:
        - key: node.cilium.io/agent-not-ready
          effect: NoSchedule
```

## Labels and Annotations

```yaml
spec:
  template:
    metadata:
      labels:
        team: platform
        environment: production
      annotations:
        cost-center: engineering
```

## Full Example

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: general-workloads
spec:
  weight: 50
  
  limits:
    cpu: "500"
    memory: 500Gi
  
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
    expireAfter: 720h  # 30 days
    budgets:
      - nodes: "20%"
      - nodes: "0"
        schedule: "0 9 * * mon-fri"
        duration: 8h
  
  template:
    metadata:
      labels:
        workload-type: general
    spec:
      nodeClassRef:
        group: karpenter.azure.com
        kind: AKSNodeClass
        name: default
      
      expireAfter: 720h
      terminationGracePeriod: 48h
      
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: [amd64]
        - key: kubernetes.io/os
          operator: In
          values: [linux]
        - key: karpenter.sh/capacity-type
          operator: In
          values: [on-demand]
        - key: karpenter.azure.com/sku-family
          operator: In
          values: [D, F]
        - key: karpenter.azure.com/sku-version
          operator: In
          values: ["3", "4", "5"]
        - key: topology.kubernetes.io/zone
          operator: In
          values: [uksouth-1, uksouth-2, uksouth-3]
```

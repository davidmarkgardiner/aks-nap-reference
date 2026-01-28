# AKS Node Auto-Provisioning (NAP) Reference

A comprehensive reference for Azure Kubernetes Service Node Auto-Provisioning, powered by Karpenter.

## What is NAP?

NAP automatically provisions the optimal VM configuration for your workloads based on pending pod requirements. It's Azure's managed implementation of [Karpenter](https://karpenter.sh/).

**Key benefits:**
- Automatic right-sizing of nodes
- Cost optimization through consolidation
- Spot instance support
- No manual node pool management

## Quick Links

- [Overview](docs/01-overview.md)
- [NodePool Configuration](docs/02-nodepools.md)
- [AKSNodeClass Configuration](docs/03-aksnodeclass.md)
- [Disruption Policies](docs/04-disruption.md)
- [Examples](examples/)

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    AKS Control Plane                         │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   Karpenter                          │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │    │
│  │  │ NodePool │  │ NodePool │  │ AKSNodeClass     │   │    │
│  │  │ (default)│  │ (spot)   │  │ (VM config)      │   │    │
│  │  └──────────┘  └──────────┘  └──────────────────┘   │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Node Provisioning                         │
│                                                              │
│  Pending Pods → Karpenter evaluates → Best VM selected      │
│                                                              │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐                  │
│  │ D-series│    │ F-series│    │ Spot    │                  │
│  │ General │    │ Compute │    │ Nodes   │                  │
│  └─────────┘    └─────────┘    └─────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

## Key Concepts

### NodePool
Defines **what** nodes can be created:
- Instance types, sizes, families
- Spot vs On-demand
- Availability zones
- Resource limits

### AKSNodeClass
Defines **how** nodes are configured:
- OS image (Ubuntu/AzureLinux)
- Disk size
- Subnet
- Kubelet settings

### Disruption
Controls **when** nodes are removed/replaced:
- Consolidation (cost optimization)
- Expiration (max node age)
- Drift (config changes)

## Prerequisites

- Azure CLI 2.76.0+
- AKS cluster with:
  - Managed identity (not service principal)
  - Azure CNI with overlay networking
  - Cilium dataplane (recommended)

## Enable NAP

```bash
# New cluster
az aks create \
  --name myCluster \
  --resource-group myRG \
  --node-provisioning-mode Auto \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --network-dataplane cilium

# Existing cluster
az aks update \
  --name myCluster \
  --resource-group myRG \
  --node-provisioning-mode Auto
```

## Limitations

- ❌ Can't enable with cluster autoscaler
- ❌ Windows node pools not supported
- ❌ IPv6 clusters not supported
- ❌ Service principals not supported
- ❌ Can't stop NAP-enabled clusters
- ❌ HTTP proxy not supported

## Comparison: NAP vs Cluster Autoscaler

| Feature | NAP (Karpenter) | Cluster Autoscaler |
|---------|-----------------|-------------------|
| Node selection | Per-pod optimal | Pre-defined pools |
| Scaling speed | Faster | Slower |
| Cost optimization | Built-in consolidation | Manual tuning |
| Spot support | Native | Separate pools |
| Configuration | NodePool CRDs | Node pool settings |
| Flexibility | High | Medium |

## File Structure

```
aks-nap-reference/
├── README.md
├── docs/
│   ├── 01-overview.md
│   ├── 02-nodepools.md
│   ├── 03-aksnodeclass.md
│   └── 04-disruption.md
└── examples/
    ├── basic-nodepool.yaml
    ├── spot-nodepool.yaml
    ├── gpu-nodepool.yaml
    └── production-setup.yaml
```

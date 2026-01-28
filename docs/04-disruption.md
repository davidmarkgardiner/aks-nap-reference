# Disruption Policies

NAP optimizes your cluster by removing/replacing underutilized nodes. This is controlled through disruption policies.

## Disruption Methods

### 1. Consolidation (Cost Optimization)

Karpenter actively reduces cost by:
- **Empty node deletion** - Remove nodes with no pods
- **Multi-node consolidation** - Delete multiple nodes, launch cheaper replacement
- **Single-node consolidation** - Replace expensive node with cheaper one

```yaml
spec:
  disruption:
    # Options: WhenEmpty, WhenEmptyOrUnderutilized
    consolidationPolicy: WhenEmptyOrUnderutilized
    
    # How long to wait before consolidating
    consolidateAfter: 1m  # or "Never"
```

| Policy | Behavior |
|--------|----------|
| `WhenEmpty` | Only consolidate empty nodes |
| `WhenEmptyOrUnderutilized` | Consolidate underutilized nodes too |

### 2. Expiration (Max Node Age)

Force node rotation after a set time:

```yaml
spec:
  disruption:
    expireAfter: 720h  # 30 days, or "Never"
```

**Why expire nodes?**
- Security (rotate credentials, patches)
- Reduce file fragmentation
- Prevent memory leaks from system processes

### 3. Drift (Config Changes)

When NodePool/AKSNodeClass changes, existing nodes become "drifted" and are replaced.

**Drifted fields include:**
- `spec.template.spec.requirements`
- `spec.template.spec.nodeClassRef`
- AKSNodeClass `spec.vnetSubnetID`

**Not drifted (behavioral fields):**
- `spec.weight`
- `spec.limits`
- `spec.disruption.*`

## Disruption Budgets

Rate-limit disruptions to prevent too many nodes being removed at once:

```yaml
spec:
  disruption:
    budgets:
      - nodes: "20%"        # Allow 20% disruption
      - nodes: "5"          # Cap at 5 nodes max
      - nodes: "0"          # Block during maintenance
        schedule: "@daily"
        duration: 10m
```

### Schedule-Based Budgets

Prevent disruptions during business hours:

```yaml
budgets:
  # No disruptions Mon-Fri 9am-5pm
  - nodes: "0"
    schedule: "0 9 * * mon-fri"
    duration: 8h
```

Weekend-only disruptions:

```yaml
budgets:
  - nodes: "50%"
    schedule: "0 0 * * 6"  # Saturday midnight
    duration: 48h
  - nodes: "0"  # Block all other times
```

## Termination Grace Period

How long to wait for pods to drain:

```yaml
spec:
  template:
    spec:
      terminationGracePeriod: 48h
```

This **overrides** pod's `terminationGracePeriodSeconds` and bypasses PDBs/do-not-disrupt annotations.

## Annotations for Control

### Prevent Pod Disruption

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        karpenter.sh/do-not-disrupt: "true"
```

### Prevent Node Disruption

```yaml
apiVersion: v1
kind: Node
metadata:
  annotations:
    karpenter.sh/do-not-disrupt: "true"
```

### Prevent All NodePool Disruption

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
spec:
  template:
    metadata:
      annotations:
        karpenter.sh/do-not-disrupt: "true"
```

## Manual Disruption

### Delete Specific Nodes

```bash
# Delete a specific node
kubectl delete node $NODE_NAME

# Delete all NAP-managed nodes
kubectl delete nodes -l karpenter.sh/nodepool

# Delete nodes from specific nodepool
kubectl delete nodes -l karpenter.sh/nodepool=$NODEPOOL_NAME
```

### Delete NodePool (Cascading Delete)

Deleting a NodePool gracefully terminates all its nodes:

```bash
kubectl delete nodepool my-nodepool
```

## Disruption Flow

```
1. Karpenter identifies disruptable node
2. Check disruption budget
3. Simulate scheduling (find replacement if needed)
4. Taint node: karpenter.sh/disrupted:NoSchedule
5. Pre-spin replacement node
6. Wait for replacement to be ready
7. Delete original node
8. Termination controller drains pods
9. Node removed
```

## Complete Example

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: production
spec:
  disruption:
    # Consolidation
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 5m
    
    # Expiration
    expireAfter: 720h  # 30 days
    
    # Budgets
    budgets:
      # Allow 20% disruption normally
      - nodes: "20%"
      
      # Cap at 5 nodes maximum
      - nodes: "5"
      
      # No disruptions during UK business hours
      - nodes: "0"
        schedule: "0 9 * * mon-fri"
        duration: 8h
      
      # Allow more disruption at night
      - nodes: "50%"
        schedule: "0 22 * * *"
        duration: 8h
  
  template:
    spec:
      # Grace period for termination
      terminationGracePeriod: 2h
```

## Monitoring Disruption

Check why nodes aren't consolidating:

```bash
kubectl get events --field-selector reason=Unconsolidatable
```

Common reasons:
- `pdb prevents pod evictions` - PodDisruptionBudget blocking
- `can't replace with a lower-priced node` - Already optimal
- `do-not-disrupt annotation` - Manual block

## Best Practices

1. **Start with `WhenEmptyOrUnderutilized`** for cost savings
2. **Set reasonable expiration** (30 days is a good default)
3. **Use budgets** to protect production during business hours
4. **Set `consolidateAfter`** to avoid thrashing (1-5 minutes)
5. **Use `do-not-disrupt`** sparingly for critical workloads
6. **Monitor events** to understand disruption behavior

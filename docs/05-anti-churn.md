# Anti-Churn Patterns

NAP/Karpenter can cause node "churning" — repeatedly creating and deleting nodes in a consolidation loop. This doc covers the root causes and fixes.

## Why Nodes Churn

| Cause | What Happens | Fix |
|-------|-------------|-----|
| **`consolidateAfter` too low** | Karpenter consolidates before cluster stabilises | Increase to 10-15m |
| **Missing resource requests** | Karpenter thinks nodes are underutilised | Enforce requests via Kyverno |
| **No PDBs** | Too many pods evicted at once → reschedule → new nodes → consolidate | Auto-inject PDBs |
| **Mixed spot + on-demand** | Spot interruptions cascade into consolidation | Separate NodePools by capacity type |
| **Same-weight NodePools** | Pools compete for same pods | Use distinct weights |
| **Bursty workloads** | Scale up → scale down → consolidate → repeat | Add `consolidateAfter` cooldown |

## Fix 1: Increase Consolidation Cooldown

The single most impactful change. Default `1m` is too aggressive for most clusters.

```yaml
spec:
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 10m  # was 1-2m — give the cluster time to settle
```

**Guideline:**
- **Production:** `10m` to `15m`
- **Standard:** `5m` to `10m`
- **Batch/Dev:** `2m` to `5m`

## Fix 2: Auto-Inject PDBs with Kyverno

Prevent Karpenter from disrupting too many pods at once. This Kyverno policy auto-creates a PDB for every Deployment with `replicas > 1`.

See: [`examples/kyverno-pdb-injection.yaml`](../examples/kyverno-pdb-injection.yaml)

### Simple Version (maxUnavailable: 1)

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: inject-pdb-simple
spec:
  failurePolicy: Ignore
  rules:
    - name: create-pdb
      match:
        any:
          - resources:
              kinds:
                - Deployment
      exclude:
        any:
          - resources:
              selector:
                matchLabels:
                  pdb.kyverno.io/inject: "false"  # opt-out label
      preconditions:
        all:
          - key: "{{ request.object.spec.replicas || `1` }}"
            operator: GreaterThan
            value: 1
      generate:
        apiVersion: policy/v1
        kind: PodDisruptionBudget
        name: "{{ request.object.metadata.name }}-pdb"
        namespace: "{{ request.object.metadata.namespace }}"
        synchronize: true
        data:
          metadata:
            labels:
              app.kubernetes.io/managed-by: kyverno
            ownerReferences:
              - apiVersion: apps/v1
                kind: Deployment
                name: "{{ request.object.metadata.name }}"
                uid: "{{ request.object.metadata.uid }}"
          spec:
            maxUnavailable: 1
            selector:
              matchLabels:
                "{{ request.object.spec.selector.matchLabels }}"
```

**Key features:**
- **Auto-generates** PDB for any Deployment with replicas > 1
- **Opt-out** with label `pdb.kyverno.io/inject: "false"`
- **ownerReferences** — PDB is garbage collected when Deployment is deleted
- **synchronize: true** — PDB stays in sync with Deployment changes
- **Skips system namespaces** (kube-system, kyverno, etc.)

### Opt-Out

Deployments that manage their own PDBs:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    pdb.kyverno.io/inject: "false"  # skip auto PDB
```

## Fix 3: Enforce Resource Requests

Without resource requests, Karpenter cannot calculate node utilisation accurately. Nodes appear "empty" and get consolidated, even when pods are using resources.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-requests
spec:
  validationFailureAction: Audit  # start with Audit, move to Enforce
  rules:
    - name: validate-requests
      match:
        any:
          - resources:
              kinds:
                - Deployment
      validate:
        message: >-
          All containers must have CPU and memory requests.
          Missing requests causes NAP/Karpenter node churn.
        foreach:
          - list: "request.object.spec.template.spec.containers[]"
            deny:
              conditions:
                any:
                  - key: "{{ element.resources.requests.cpu || '' }}"
                    operator: Equals
                    value: ""
                  - key: "{{ element.resources.requests.memory || '' }}"
                    operator: Equals
                    value: ""
```

**Rollout plan:**
1. Deploy with `Audit` mode — see violations in policy reports
2. Fix non-compliant Deployments
3. Switch to `Enforce` mode

## Fix 4: Separate Spot and On-Demand Pools

Don't mix `capacity-type: [spot, on-demand]` in one NodePool. Spot interruptions trigger pod rescheduling, which cascades into consolidation decisions.

```yaml
# ❌ BAD — mixed capacity types cause churn
requirements:
  - key: karpenter.sh/capacity-type
    operator: In
    values: [spot, on-demand]

# ✅ GOOD — separate pools
---
# Pool 1: On-demand for stable workloads
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: stable
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: [on-demand]
---
# Pool 2: Spot for batch/tolerant workloads
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: spot-batch
spec:
  template:
    spec:
      taints:
        - key: capacity-type
          value: spot
          effect: NoSchedule
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: [spot]
```

## Fix 5: Business Hours Protection

Block consolidation during business hours entirely:

```yaml
disruption:
  budgets:
    # No disruption Mon-Fri 8am-6pm UK
    - nodes: "0"
      schedule: "0 8 * * mon-fri"
      duration: 10h
    # Allow 20% outside business hours
    - nodes: "20%"
```

## Fix 6: do-not-disrupt for Critical Pods

For pods that should never be moved by consolidation:

```yaml
metadata:
  annotations:
    karpenter.sh/do-not-disrupt: "true"
```

Use sparingly — too many do-not-disrupt annotations prevent Karpenter from optimising at all.

## Monitoring Churn

```bash
# Watch for consolidation events
kubectl get events --field-selector reason=Unconsolidatable -A
kubectl get events --field-selector reason=DisruptionBlocked -A

# Check Karpenter logs for consolidation decisions
kubectl logs -n kube-system -l app.kubernetes.io/name=karpenter -c controller \
  | grep -E "consolidat|disrupt|replace"

# Count node creations/deletions in last hour
kubectl get events -A --field-selector reason=Registered \
  --sort-by='.lastTimestamp' | tail -20
```

## Recommended Production Config

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: production
spec:
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 10m      # generous cooldown
    expireAfter: 720h           # 30 days
    budgets:
      - nodes: "10%"           # never more than 10%
      - nodes: "0"             # block during business hours
        schedule: "0 8 * * mon-fri"
        duration: 10h
  template:
    spec:
      terminationGracePeriod: 2h
```

Combined with:
- ✅ Kyverno auto-PDB injection
- ✅ Kyverno resource requests enforcement
- ✅ Separate spot/on-demand NodePools
- ✅ Monitoring + alerting on churn rate

## Known Issues

- [Karpenter #1851](https://github.com/kubernetes-sigs/karpenter/issues/1851) — Consolidation loop when scaling replicas with same-weight NodePools
- Multiple NodePools with identical weights competing for the same pods
- `consolidateAfter` resets on every evaluation, not just state changes

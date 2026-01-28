# Flux + Kustomize NAP Configuration Example

Manage NAP (Karpenter) configuration across multiple clusters using Flux and Kustomize with `patchesJson6902`.

## Structure

```
flux-example/
├── infrastructure/
│   └── nap/
│       └── base/                          # 👈 Source of truth
│           ├── kustomization.yaml
│           ├── nodepool-default.yaml
│           └── aksnodeclass-default.yaml
│
└── clusters/
    ├── prod-uksouth/                      # Uses base as-is
    │   └── nap/
    │       └── kustomization.yaml
    │
    └── test-uksouth/                      # 👈 patchesJson6902 overrides
        └── nap/
            └── kustomization.yaml
```

## patchesJson6902 Format

Uses RFC 6902 JSON Patch operations with explicit target specification.

### Example: Test Cluster Kustomization

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../../infrastructure/nap/base

patchesJson6902:
  - target:
      group: karpenter.sh
      version: v1
      kind: NodePool
      name: default
    patch: |-
      - op: replace
        path: /spec/disruption/consolidationPolicy
        value: "WhenEmpty"
      - op: replace
        path: /spec/disruption/consolidateAfter
        value: "10m"
      - op: replace
        path: /spec/disruption/budgets
        value:
          - nodes: "10%"
          - nodes: "2"
          - nodes: "0"
            schedule: "0 8 * * mon-fri"
            duration: 10h
```

## Target Specification

```yaml
target:
  group: karpenter.sh           # API group
  version: v1                   # API version
  kind: NodePool                # Resource kind
  name: default                 # Resource name
  namespace: kube-system        # Optional: namespace
```

### Common Targets

| Resource | group | version | kind |
|----------|-------|---------|------|
| NodePool | `karpenter.sh` | `v1` | `NodePool` |
| AKSNodeClass | `karpenter.azure.com` | `v1beta1` | `AKSNodeClass` |

## JSON Patch Operations (RFC 6902)

| Operation | Use Case | Example |
|-----------|----------|---------|
| `replace` | Change existing value | `op: replace` `path: /spec/field` `value: "new"` |
| `add` | Add new field or array item | `op: add` `path: /spec/new` `value: "val"` |
| `remove` | Delete a field | `op: remove` `path: /spec/field` |
| `copy` | Copy value from another path | `op: copy` `from: /spec/a` `path: /spec/b` |
| `move` | Move value to another path | `op: move` `from: /spec/a` `path: /spec/b` |
| `test` | Assert value exists | `op: test` `path: /spec/field` `value: "expected"` |

### Path Syntax

- `/spec/disruption/consolidateAfter` — nested field
- `/spec/disruption/budgets/0` — first array item (index)
- `/spec/disruption/budgets/-` — append to end of array

## Test Cluster Changes

| Setting | Base | Test (Patched) |
|---------|------|----------------|
| `consolidationPolicy` | `WhenEmptyOrUnderutilized` | `WhenEmpty` |
| `consolidateAfter` | `1m` | `10m` |
| `budgets[0].nodes` | `20%` | `10%` |
| Business hours protection | None | Mon-Fri 8am-6pm |

## Using External Patch Files

```yaml
patchesJson6902:
  - target:
      group: karpenter.sh
      version: v1
      kind: NodePool
      name: default
    path: patches/nodepool.json
```

**`patches/nodepool.json`**
```json
[
  {"op": "replace", "path": "/spec/disruption/consolidateAfter", "value": "10m"},
  {"op": "replace", "path": "/spec/disruption/consolidationPolicy", "value": "WhenEmpty"}
]
```

## Common Patches

### Slow down consolidation

```yaml
- op: replace
  path: /spec/disruption/consolidateAfter
  value: "10m"
```

### Change to WhenEmpty only

```yaml
- op: replace
  path: /spec/disruption/consolidationPolicy
  value: "WhenEmpty"
```

### Replace entire budgets array

```yaml
- op: replace
  path: /spec/disruption/budgets
  value:
    - nodes: "10%"
    - nodes: "0"
      schedule: "0 8 * * mon-fri"
      duration: 10h
```

### Append to existing budgets array

```yaml
- op: add
  path: /spec/disruption/budgets/-
  value:
    nodes: "0"
    schedule: "0 8 * * mon-fri"
    duration: 10h
```

### Disable consolidation entirely

```yaml
- op: replace
  path: /spec/disruption/consolidateAfter
  value: "Never"
```

### Add annotation

```yaml
- op: add
  path: /spec/template/metadata/annotations/karpenter.sh~1do-not-disrupt
  value: "true"
```

Note: `/` in keys must be escaped as `~1`

## Preview Changes

```bash
# Build and see final output
kustomize build clusters/test-uksouth/nap

# Compare prod vs test
diff <(kustomize build clusters/prod-uksouth/nap) \
     <(kustomize build clusters/test-uksouth/nap)

# Validate syntax
kustomize build clusters/test-uksouth/nap > /dev/null && echo "Valid"

# Dry-run apply
kustomize build clusters/test-uksouth/nap | kubectl apply --dry-run=server -f -
```

## Adding a New Cluster

1. Create cluster directory:
   ```bash
   mkdir -p clusters/new-cluster/nap
   ```

2. Create `kustomization.yaml` with `patchesJson6902`:
   ```yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   resources:
     - ../../../infrastructure/nap/base
   patchesJson6902:
     - target:
         group: karpenter.sh
         version: v1
         kind: NodePool
         name: default
       patch: |-
         - op: replace
           path: /spec/disruption/consolidateAfter
           value: "5m"
   ```

3. Create Flux Kustomization resource

4. Commit and push — Flux reconciles automatically

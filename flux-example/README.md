# Flux + Kustomize NAP Configuration Example

Manage NAP (Karpenter) configuration across multiple clusters using Flux and Kustomize with JSON Patch.

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
    └── test-uksouth/                      # 👈 JSON Patch overrides
        └── nap/
            └── kustomization.yaml
```

## JSON Patch Method

We use JSON Patch (`op: replace`, `op: add`, `op: remove`) for surgical changes. This is cleaner than strategic merge patches for specific field overrides.

### Example: Test Cluster Kustomization

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../../infrastructure/nap/base

patches:
  - target:
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

## JSON Patch Operations

| Operation | Use Case | Example |
|-----------|----------|---------|
| `replace` | Change existing value | `op: replace` `path: /spec/disruption/consolidateAfter` `value: "10m"` |
| `add` | Add new field or array item | `op: add` `path: /spec/disruption/budgets/-` `value: {nodes: "0"}` |
| `remove` | Delete a field | `op: remove` `path: /spec/template/spec/taints` |

### Path Syntax

- `/spec/disruption/consolidateAfter` — nested field
- `/spec/disruption/budgets/0` — first array item
- `/spec/disruption/budgets/-` — append to array

## Test Cluster Changes

| Setting | Base | Test (Patched) |
|---------|------|----------------|
| `consolidationPolicy` | `WhenEmptyOrUnderutilized` | `WhenEmpty` |
| `consolidateAfter` | `1m` | `10m` |
| `budgets[0].nodes` | `20%` | `10%` |
| Business hours protection | None | Mon-Fri 8am-6pm |

## Preview Changes

```bash
# Build and see final output
kustomize build clusters/test-uksouth/nap

# Compare prod vs test
diff <(kustomize build clusters/prod-uksouth/nap) \
     <(kustomize build clusters/test-uksouth/nap)

# Flux CLI preview
flux build kustomization nap-config --path ./clusters/test-uksouth/nap
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

### Add business hours protection

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

### Add do-not-disrupt annotation to NodePool

```yaml
- op: add
  path: /spec/template/metadata/annotations
  value:
    karpenter.sh/do-not-disrupt: "true"
```

## Adding a New Cluster

1. Create cluster directory:
   ```bash
   mkdir -p clusters/new-cluster/nap
   ```

2. Create `kustomization.yaml`:
   ```yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   resources:
     - ../../../infrastructure/nap/base
   patches:
     - target:
         kind: NodePool
         name: default
       patch: |-
         - op: replace
           path: /spec/disruption/consolidateAfter
           value: "5m"
   ```

3. Create Flux Kustomization resource

4. Commit and push — Flux reconciles automatically

## Validation

```bash
# Validate kustomization syntax
kustomize build clusters/test-uksouth/nap > /dev/null && echo "Valid"

# Apply dry-run
kustomize build clusters/test-uksouth/nap | kubectl apply --dry-run=server -f -
```

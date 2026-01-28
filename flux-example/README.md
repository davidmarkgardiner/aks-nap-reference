# Flux + Kustomize NAP Configuration Example

This example shows how to manage NAP (Karpenter) configuration across multiple clusters using Flux and Kustomize.

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
    ├── prod-uksouth/                      # 👈 Uses base as-is
    │   ├── nap/
    │   │   └── kustomization.yaml
    │   └── nap-kustomization.yaml         # Flux Kustomization
    │
    └── test-uksouth/                      # 👈 Patches base config
        ├── nap/
        │   ├── kustomization.yaml
        │   └── patches/
        │       ├── nodepool-slow-consolidation.yaml
        │       └── aksnodeclass-test-tags.yaml
        └── nap-kustomization.yaml         # Flux Kustomization
```

## How It Works

1. **Base config** (`infrastructure/nap/base/`) contains the default NodePool and AKSNodeClass
2. **Prod cluster** references base directly with no changes
3. **Test cluster** references base but applies patches to slow down consolidation

## Test Cluster Changes

The test cluster patch (`nodepool-slow-consolidation.yaml`) makes these changes:

| Setting | Base | Test Cluster |
|---------|------|--------------|
| `consolidationPolicy` | `WhenEmptyOrUnderutilized` | `WhenEmpty` |
| `consolidateAfter` | `1m` | `10m` |
| `budgets[0].nodes` | `20%` | `10%` |
| Business hours protection | None | Mon-Fri 8am-6pm |

## Preview Changes

Before applying, preview what Flux will deploy:

```bash
# Using Flux CLI
flux build kustomization nap-config \
  --path ./clusters/test-uksouth/nap

# Using kustomize directly
kustomize build clusters/test-uksouth/nap

# Compare prod vs test
diff <(kustomize build clusters/prod-uksouth/nap) \
     <(kustomize build clusters/test-uksouth/nap)
```

## Apply to Cluster

```bash
# If using Flux GitOps (recommended)
# Just push to git - Flux will reconcile automatically

# Manual apply for testing
kustomize build clusters/test-uksouth/nap | kubectl apply -f -
```

## Adding a New Cluster

1. Copy an existing cluster folder:
   ```bash
   cp -r clusters/prod-uksouth clusters/new-cluster
   ```

2. Update `kustomization.yaml` with any cluster-specific patches

3. Update `nap-kustomization.yaml` with the correct path

4. Commit and push - Flux handles the rest

## Rollout Strategy

1. Test changes in `test-uksouth` first
2. Monitor for 24-48 hours
3. If stable, remove patches from test OR apply same patches to prod
4. Gradually roll to other clusters

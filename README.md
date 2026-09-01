# OpenShift ACM GitOps as Code

Hub-side ACM governance that installs OpenShift operators on managed clusters through Kustomize.

This is **not** a raw OLM bundle. Each overlay renders a hub `Policy` that embeds:

1. A `ConfigurationPolicy` that creates the operator namespace.
2. An `OperatorPolicy` that creates the OLM `Subscription` and `OperatorGroup` on selected clusters.

Hub objects are namespaced in `open-cluster-management-global-set` and placed onto the `global` ManagedClusterSet.

## Layout

```
acm-gitops/governance/install-operator/
  base/                         # generic skeleton — do not apply directly
    policy.yaml
    placement.yaml
    placementbinding.yaml
  overlays/
    gitops/                     # OpenShift GitOps Operator
    devspaces/                  # OpenShift Dev Spaces Operator
```

`base/` uses placeholder values (`placeholder-operator`, `openshift-operators`). Overlays JSON-patch the **top-level** `Policy`, `Placement`, and `PlacementBinding`. Nested `OperatorPolicy` is not a Kustomize resource; patch it through `/spec/policy-templates/1/objectDefinition/...`.

## Prerequisites

- OpenShift hub with ACM (OperatorPolicy enabled on managed clusters).
- `oc` (or `kustomize`) locally.
- Managed clusters in the `global` ClusterSet (already bound to `open-cluster-management-global-set`).

## Preview

```bash
oc kustomize acm-gitops/governance/install-operator/overlays/gitops
oc kustomize acm-gitops/governance/install-operator/overlays/devspaces
```

| Overlay | Policy / Placement / Binding | Package | Channel | Namespace |
|---|---|---|---|---|
| `overlays/gitops` | `install-gitops` | `openshift-gitops-operator` | `latest` | `openshift-gitops-operator` |
| `overlays/devspaces` | `install-devspaces` | `devspaces` | `stable` | `openshift-devspaces` |

OperatorGroups omit `targetNamespaces` (AllNamespaces). Both overlays can be applied to the same hub; names do not collide.

## Apply

```bash
oc apply -k acm-gitops/governance/install-operator/overlays/gitops
oc apply -k acm-gitops/governance/install-operator/overlays/devspaces
```

Check hub placement and policy status:

```bash
oc get policy,placement,placementbinding -n open-cluster-management-global-set
oc get placementdecision -n open-cluster-management-global-set
```

On a managed cluster, confirm OLM objects:

```bash
oc get ns openshift-gitops-operator
oc get subscription,operatorgroup,csv -n openshift-gitops-operator
```

## Add another operator

1. Copy an existing overlay directory, for example `overlays/gitops` → `overlays/my-operator`.
2. Keep `resources: [../../base]` and `target.name: install-operator` (the **base** names).
3. Patch unique names plus subscription fields. Required Policy JSON pointers:

   | Path | Purpose |
   |---|---|
   | `/metadata/name` | Hub Policy name (must be unique per overlay) |
   | `/spec/policy-templates/0/.../metadata/name` | ConfigurationPolicy name |
   | `/spec/policy-templates/0/.../objectDefinition/metadata/name` | Namespace to create |
   | `/spec/policy-templates/1/objectDefinition/metadata/name` | OperatorPolicy name |
   | `/spec/policy-templates/1/.../operatorGroup/{name,namespace}` | OperatorGroup |
   | `/spec/policy-templates/1/.../subscription/{name,namespace,channel}` | OLM Subscription |

4. Mirror the new name on `Placement` (`/metadata/name`) and `PlacementBinding` (`/metadata/name`, `/placementRef/name`, `/subjects/0/name`).
5. Preview with `oc kustomize`, then `oc apply -k`.

Do not GitOps-manage ACM’s built-in Placement named `global`.

## OperatorPolicy vs OLM

| OLM field | Set this on OperatorPolicy | Do not set |
|---|---|---|
| `installPlanApproval: Automatic` | `upgradeApproval: Automatic` | `subscription.installPlanApproval` (rejected) |
| `installPlanApproval: Manual` | `upgradeApproval: None` and `versions: [...]` | same |
| Initial CSV pin | `subscription.startingCSV` | — |
| OperatorGroup scope | Omit `targetNamespaces` for AllNamespaces; set the list for OwnNamespace | — |

`upgradeApproval` controls **upgrades**. An `enforce` policy still approves the initial InstallPlan when the CSV is allowed.

### Pin a version (production overlay snippet)

Add these ops to the Policy patch (template index `1` is the OperatorPolicy):

```yaml
- op: replace
  path: /spec/policy-templates/1/objectDefinition/spec/upgradeApproval
  value: None
- op: add
  path: /spec/policy-templates/1/objectDefinition/spec/subscription/startingCSV
  value: my-operator.v1.2.3
- op: add
  path: /spec/policy-templates/1/objectDefinition/spec/versions
  value:
    - my-operator.v1.2.3
```

Omit `versions` when `upgradeApproval` is `Automatic` and any CSV is acceptable.

## OwnNamespace operators

Add `targetNamespaces` on the OperatorGroup (same namespace as the Subscription):

```yaml
- op: add
  path: /spec/policy-templates/1/objectDefinition/spec/operatorGroup/targetNamespaces
  value:
    - openshift-devspaces
```

## Target a different ClusterSet

Default Placement uses `clusterSets: [global]` in `open-cluster-management-global-set`. To use another set, patch **Policy, Placement, and PlacementBinding** `metadata.namespace` and Placement `spec.clusterSets`, and ensure a `ManagedClusterSetBinding` exists in that namespace.

# fleet-clusters

Declarative cluster lifecycle state for Red Hat OpenShift Partner Labs.

This repo holds per-cluster specifications as Kustomize overlays. It is the data plane counterpart to [fleet](https://github.com/redhat-openshift-partner-labs/fleet), which contains the Tekton pipelines, Python CLI tools, and hub configuration that act on these definitions.

Commits to this repo trigger pipelines on the hub cluster via webhook:
- Adding a cluster directory under `provision/` triggers the **pre-provision** and **provision** pipelines.
- Adding a sentinel file under `deprovision/` triggers the **deprovision** pipeline.
- After successful deprovision, a GitHub Action archives the cluster spec.

## Repo Layout

```
cluster-templates/
  <provider>/                 Cloud provider (aws, gcp, azure)
    <variant>/base/           Cluster topology variant
      crossplane/             Provider-specific IAM/identity resources
      hive/                   Namespace, ClusterDeployment, MachinePool,
                              ManagedCluster, KlusterletAddonConfig, install-config Secret

  Variants:
    standard/                 3 control plane + 3 worker nodes (HA)
    sno/                      Single-node OpenShift (1 CP, 0 workers)
    compact/                  3 schedulable control plane nodes, 0 workers

  aws-ha/base/                Legacy base (kept for existing overlay compatibility)

provision/
  <cluster-id>/               Active cluster spec (Kustomize overlay on a template base)
    kustomization.yaml        References crossplane/ and hive/ subdirectories
    crossplane/
      kustomization.yaml
      patches/                IAM/identity resource name patches
    hive/
      kustomization.yaml
      patches/                Cluster name, region, instance types, replicas, tier

deprovision/
  <cluster-id>                Sentinel file — presence triggers deprovision pipeline

archive/
  <cluster-id>/               Preserved cluster spec after deprovision (moved by GitHub Action)

.github/workflows/            GitHub Action for post-deprovision archive automation
```

## Cluster Lifecycle

### Provisioning

1. Create a cluster directory under `provision/` using the scaffold tool (from the fleet repo):
   ```bash
   fleet-scaffold-cluster --name <cluster-id> --region us-east-1 --tier base
   ```
   Or copy an existing overlay and replace every occurrence of the old cluster name. Point the overlay's `resources:` at the appropriate template base (e.g., `cluster-templates/aws/standard/base/hive`).

2. Commit and push. The webhook fires against the hub's Tekton EventListener, which triggers the pre-provision pipeline followed by the provision pipeline.

3. The provision pipeline creates Crossplane IAM credentials, validates inputs, applies Hive CRs, waits for the cluster to become ready, and chains into the post-provision pipeline (SSL, OAuth, RBAC, workloads).

### Deprovisioning

1. Create a sentinel file under `deprovision/`:
   ```bash
   touch deprovision/<cluster-id>
   ```

2. Commit and push. The deprovision pipeline deletes cluster CRs in controlled order, waits for Hive uninstall, and cleans up hub artifacts.

3. On successful deprovision, the pipeline emits a GitHub deployment status. The GitHub Action archives the cluster spec:
   ```
   git mv provision/<cluster-id> archive/<cluster-id>
   rm deprovision/<cluster-id>
   ```

### One Cluster Per Push

The EventListener CEL interceptor extracts one cluster name per push. If a single push adds multiple cluster directories, only the first triggers the pipeline. Push one cluster per commit.

## Cluster Spec Structure

Each cluster overlay patches a `cluster-templates/<provider>/<variant>/base/` Kustomize base with cluster-specific values.

### Crossplane Patches (`crossplane/patches/`)

Provider-specific identity and credential resource patches:

| Provider | Resources |
|----------|-----------|
| AWS | IAM User, Policy, UserPolicyAttachment, AccessKey |
| GCP | ServiceAccount, ServiceAccountKey, ProjectIAMMember |
| Azure | UserAssignedIdentity, RoleAssignment, FederatedIdentityCredential |

### Hive Patches (`hive/patches/`)

| File | Purpose |
|------|---------|
| `namespace.yaml` | Namespace name |
| `clusterdeployment.yaml` | Name, namespace, region, secret references |
| `machinepool-worker.yaml` | Name, namespace, instance type, zones, replicas |
| `managedcluster.yaml` | Name, tier/environment/region labels |
| `klusterletaddonconfig.yaml` | Name, namespace, cluster references |
| `install-config-meta.yaml` | Secret metadata (name, namespace) |
| `install-config.yaml` | Full install-config (strategic merge patch) |

## Base Template Defaults

All providers share: base domain `openshiftpartnerlabs.com`, network type `OVNKubernetes`, image set `img4.20.10-x86-64-appsub`.

### AWS (`cluster-templates/aws/`)

| Setting | Standard | SNO | Compact |
|---------|----------|-----|---------|
| Region | `us-east-1` | `us-east-1` | `us-east-1` |
| Zones | `us-east-1a`, `1b`, `1c` | `us-east-1a` | `us-east-1a`, `1b`, `1c` |
| CP instance type | `m8i.2xlarge` | `m8i.2xlarge` | `m8i.2xlarge` |
| CP replicas | `3` | `1` | `3` |
| Worker instance type | `m8i.2xlarge` | — | — |
| Worker replicas | `3` | `0` | `0` |
| Worker root volume | `100 GB gp3, 4000 IOPS` | — | — |
| MachinePool type | `m5.2xlarge` | — | — |

### GCP (`cluster-templates/gcp/`)

| Setting | Standard | SNO | Compact |
|---------|----------|-----|---------|
| Region | `us-central1` | `us-central1` | `us-central1` |
| Zones | `us-central1-a`, `-b`, `-c` | `us-central1-a` | `us-central1-a`, `-b`, `-c` |
| CP instance type | `n2-standard-8` | `n2-standard-8` | `n2-standard-8` |
| CP replicas | `3` | `1` | `3` |
| Worker instance type | `n2-standard-8` | — | — |
| Worker replicas | `3` | `0` | `0` |
| Worker root volume | `128 GB pd-ssd` | — | — |
| MachinePool type | `n2-standard-8` | — | — |

### Azure (`cluster-templates/azure/`)

| Setting | Standard | SNO | Compact |
|---------|----------|-----|---------|
| Region | `eastus` | `eastus` | `eastus` |
| Zones | `1`, `2`, `3` | `1` | `1`, `2`, `3` |
| CP instance type | `Standard_D8s_v5` | `Standard_D8s_v5` | `Standard_D8s_v5` |
| CP replicas | `3` | `1` | `3` |
| Worker instance type | `Standard_D8s_v5` | — | — |
| Worker replicas | `3` | `0` | `0` |
| Worker root volume | `128 GB Premium_LRS` | — | — |
| MachinePool type | `Standard_D8s_v5` | — | — |

## Tier Labels

Set on `ManagedCluster` via `hive/patches/managedcluster.yaml`:

| Tier | Day-2 Workloads |
|------|-----------------|
| `base` | Baseline operators, monitoring, NetworkPolicies |
| `virt` | Base + OpenShift Virtualization (CNV) |
| `ai` | Base + GPU Operator + OpenShift AI |

## Cross-File Coordination

Values that must stay in sync across multiple patch files:

| Value | Must match in |
|-------|--------------|
| Cluster name | Namespace, ClusterDeployment, MachinePool, ManagedCluster, KlusterletAddonConfig, Secret, all Crossplane resources, install-config `metadata.name` |
| Region | ClusterDeployment `spec.platform.aws.region`, install-config `platform.aws.region` |
| Zones | install-config `controlPlane.platform.aws.zones[]`, install-config `compute[0].platform.aws.zones[]` (must be valid for the region) |
| Install-config secret name | ClusterDeployment `installConfigSecretRef.name`, Secret `metadata.name` (must be `<cluster-name>-install-config`) |
| IAM user name | User `metadata.name`, UserPolicyAttachment `userRef.name`, AccessKey `userRef.name` |
| IAM policy name | Policy `metadata.name`, UserPolicyAttachment `policyArnRef.name` |
| Credential namespace | AccessKey `writeConnectionSecretToRef.namespace` (must match cluster namespace) |

## Common Failures

| Symptom | Root Cause | Fix |
|---------|-----------|-----|
| `Zone does not exist in region` | Region changed but zones still reference old region | Update zones in install-config to match new region |
| ClusterDeployment stuck provisioning | `installConfigSecretRef.name` mismatch | Ensure both use `<cluster-name>-install-config` |
| ClusterDeployment stuck provisioning | Region mismatch between ClusterDeployment and install-config | Set both to the same region |
| AWS credentials not found | AccessKey `writeConnectionSecretToRef.namespace` mismatch | Set namespace to cluster name |
| MachinePool not joining | `clusterDeploymentRef.name` mismatch | Patch MachinePool to match ClusterDeployment name |
| IAM policy not attached | Policy name mismatch in UserPolicyAttachment | Ensure both use `<cluster-name>-openshift4installerpolicy` |

## Validating a Cluster Spec

```bash
# Validate a cluster overlay
kustomize build provision/<cluster-id>

# Validate a base template
kustomize build cluster-templates/<provider>/<variant>/base/
```

This renders the full set of Kubernetes manifests. Verify names, regions, and cross-references are consistent before pushing.

## Related

- [fleet](https://github.com/redhat-openshift-partner-labs/fleet) — Tekton pipelines, Python CLI tools, hub configuration, and workload overlays that operate on cluster specs defined here.

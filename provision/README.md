# Cluster Overlays

Each subdirectory defines a cluster as a Kustomize overlay on a template base under `cluster-templates/<provider>/<variant>/base/`. Multiple files reference the same values (region, cluster name, instance types), and they **must stay in sync** — changing a value in one file but not the others causes install failures.

For base template defaults, cross-file coordination requirements, common failures, and tier labels, see the [main README](../README.md).

## Directory Structure

```
provision/<cluster-name>/
├── kustomization.yaml                        # References crossplane/ and hive/
├── crossplane/
│   ├── kustomization.yaml                    # target+path refs only
│   └── patches/                              # Provider-specific identity patches
└── hive/
    ├── kustomization.yaml                    # target+path refs only
    └── patches/
        ├── install-config.yaml               # Full install-config (strategic merge)
        ├── install-config-meta.yaml           # Secret metadata (name, namespace)
        ├── namespace.yaml                    # Namespace name
        ├── clusterdeployment.yaml            # Name, namespace, region, secret refs
        ├── machinepool-worker.yaml           # Name, namespace, type, zones, replicas
        ├── managedcluster.yaml               # Name, tier/environment labels
        └── klusterletaddonconfig.yaml        # Name, namespace, cluster refs
```

### Crossplane Patches by Provider

| Provider | Patch files | Resources |
|----------|------------|-----------|
| AWS | `user.yaml`, `policy.yaml`, `policy-attachment.yaml`, `access-key.yaml` | IAM User, Policy, UserPolicyAttachment, AccessKey |
| GCP | `service-account.yaml`, `service-account-key.yaml`, `project-iam-member.yaml` | ServiceAccount, ServiceAccountKey, ProjectIAMMember |
| Azure | `user-assigned-identity.yaml`, `role-assignment.yaml`, `federated-identity-credential.yaml` | UserAssignedIdentity, RoleAssignment, FederatedIdentityCredential |

### Hive Patches

All providers share the same hive patch files. The `kustomization.yaml` files contain only `target:` + `path:` references — no inline patch blocks.

| File | Patch type | Purpose |
|------|-----------|---------|
| `install-config.yaml` | Strategic merge | Full install-config with infra settings |
| All others | JSON patch | Resource names, namespace, region, labels, secret refs |

## Pushing Clusters

The EventListener CEL interceptor extracts one cluster name per push. If a single commit or push adds multiple cluster directories, only the first one triggers the pre-provision pipeline — the rest are silently dropped.

**Push one cluster per commit.** If you need to provision multiple clusters, push them as separate commits or in separate pushes.

## Creating a New Cluster

1. Choose a base template: `cluster-templates/<provider>/<variant>/base/`
2. Create the overlay directory under `provision/<name>/`
3. Replace every occurrence of `cluster-placeholder` with the actual cluster name
4. Set the region, zones, instance types, and replicas in the install-config and patches
5. Validate: `kustomize build provision/<name>`

### Scaffold Tool

The fleet repo provides `fleet-scaffold-cluster` to generate overlays automatically:

```bash
fleet-scaffold-cluster --name <cluster-id> --region us-east-1 --tier base
```

### Manual Creation

Copy an existing overlay and replace every occurrence of the old cluster name. Point the overlay's `resources:` at the appropriate template base.

#### Checklist

1. Create `provision/<name>/kustomization.yaml` — reference `crossplane/` and `hive/`
2. Create `provision/<name>/crossplane/kustomization.yaml` — set `resources:` to the provider-specific base, replace all `cluster-placeholder` names
3. Create `provision/<name>/hive/kustomization.yaml` — set `resources:` to the provider/variant base, replace all `cluster-placeholder` names, set region and tier
4. Create `provision/<name>/hive/patches/install-config.yaml` — set instance types, replicas, zones, region
5. Validate both subdirectories:
   ```bash
   kustomize build provision/<name>/crossplane
   kustomize build provision/<name>/hive
   kustomize build provision/<name>
   ```

### Cluster Name

Every file patches the base placeholder `cluster-placeholder` to the actual cluster name. The name appears in metadata, spec references, and cross-resource refs. All must match.

**Hive resources:** Namespace `/metadata/name`, ClusterDeployment `/metadata/name` + `/metadata/namespace` + `/spec/clusterName`, MachinePool `/metadata/name` + `/metadata/namespace` + `/spec/clusterDeploymentRef/name`, ManagedCluster `/metadata/name`, KlusterletAddonConfig `/metadata/name` + `/metadata/namespace` + `/spec/clusterName` + `/spec/clusterNamespace`, Secret `/metadata/name` + `/metadata/namespace`. Also: ClusterDeployment `/spec/provisioning/installConfigSecretRef/name` (must be `<name>-install-config`) and `/spec/provisioning/sshPrivateKeySecretRef/name` (must be `<name>-ssh-key`). The install-config `metadata.name` must also match.

**Crossplane resources (by provider):**

| Provider | Fields to patch |
|----------|----------------|
| AWS | User `/metadata/name` (`<name>-ocp-installer`), Policy `/metadata/name` + `/spec/forProvider/name`, UserPolicyAttachment `/metadata/name` + refs to user and policy, AccessKey `/metadata/name` + `/spec/forProvider/userRef/name` + `/spec/writeConnectionSecretToRef/namespace` |
| GCP | ServiceAccount `/metadata/name` (`<name>-ocp-installer`), ServiceAccountKey `/metadata/name` + ref to SA + `/spec/writeConnectionSecretToRef/namespace`, ProjectIAMMember `/metadata/name` + ref to SA |
| Azure | UserAssignedIdentity `/metadata/name` (`<name>-ocp-installer`), RoleAssignment `/metadata/name` + ref to identity, FederatedIdentityCredential `/metadata/name` + ref to identity + subject namespace |

## Common Changes

### Region and Availability Zones

Region appears in the ClusterDeployment and install-config — both must agree. Zones must be valid for the chosen region.

| File | Field |
|------|-------|
| `hive/kustomization.yaml` | ClusterDeployment `/spec/platform/<provider>/region` |
| `hive/patches/install-config.yaml` | `platform.<provider>.region` |
| `hive/patches/install-config.yaml` | `controlPlane.platform.<provider>.zones[]` |
| `hive/patches/install-config.yaml` | `compute[0].platform.<provider>.zones[]` |

If you change the region, update **all four**. The `<provider>` key is `aws`, `gcp`, or `azure` depending on the template base.

### Instance Types

Instance types are set **independently** for masters, workers (install-config), and the MachinePool. They do not auto-sync.

| Component | File | Field |
|-----------|------|-------|
| Masters | `hive/patches/install-config.yaml` | `controlPlane.platform.<provider>.type` |
| Workers (install-config) | `hive/patches/install-config.yaml` | `compute[0].platform.<provider>.type` |
| Workers (MachinePool) | base `machinepool-worker.yaml` | `/spec/platform/<provider>/type` |

The MachinePool governs day-2 scaling. If it specifies a different instance type than the install-config, new nodes added after install will use the MachinePool type.

To override the MachinePool instance type, add a patch in `hive/kustomization.yaml`:

```yaml
- target:
    kind: MachinePool
    name: cluster-placeholder-worker
  patch: |
    - op: replace
      path: /spec/platform/<provider>/type
      value: <instance-type>
```

### Node Count (Replicas)

Replicas are set in the install-config and independently in the MachinePool.

| Component | File | Field |
|-----------|------|-------|
| Masters | `hive/patches/install-config.yaml` | `controlPlane.replicas` |
| Workers (install-config) | `hive/patches/install-config.yaml` | `compute[0].replicas` |
| Workers (MachinePool) | base `machinepool-worker.yaml` | `/spec/replicas` |

For SNO (single-node OpenShift): use the `sno` variant base, which sets `controlPlane.replicas: 1` and `compute[0].replicas: 0`.

For compact clusters: use the `compact` variant base, which sets `controlPlane.replicas: 3` (schedulable) and `compute[0].replicas: 0`.

### Tier Label

Set on ManagedCluster in `hive/patches/managedcluster.yaml` or `hive/kustomization.yaml`.

```yaml
- target:
    kind: ManagedCluster
    name: cluster-placeholder
  patch: |
    - op: add
      path: /metadata/labels/tier
      value: base    # base | virt | ai
```

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

Declarative cluster lifecycle state for Red Hat OpenShift Partner Labs. Each cluster is a Kustomize overlay under `provision/` that patches a base template in `cluster-templates/<provider>/<variant>/base/`. Commits trigger Tekton pipelines on a hub cluster via webhook. The companion [fleet](https://github.com/redhat-openshift-partner-labs/fleet) repo contains the pipelines and CLI tools that act on these definitions.

## Key Commands

```bash
# Validate a cluster overlay (the primary "build" check)
kustomize build provision/<cluster-id>

# Validate sub-components individually
kustomize build provision/<cluster-id>/crossplane
kustomize build provision/<cluster-id>/hive

# Validate a base template
kustomize build cluster-templates/<provider>/<variant>/base/

# Run pre-commit hooks (gitleaks secret scanning)
pre-commit run --all-files

# Test GitHub Action locally with act
act deployment_status
```

## Architecture

### Lifecycle flow

1. **Provision**: add cluster dir under `provision/` → push → webhook triggers Tekton pre-provision + provision pipelines on hub
2. **Deprovision**: add sentinel file under `deprovision/<cluster-id>` → push → webhook triggers deprovision pipeline
3. **Archive**: on successful deprovision, GitHub Action moves `provision/<id>` to `archive/<id>` and removes the sentinel

### One cluster per push

The EventListener CEL interceptor extracts one cluster name per push. Multiple cluster directories in one push means only the first triggers the pipeline.

### Template hierarchy

```
cluster-templates/
  <provider>/          # aws, gcp, azure (also aws-ha as legacy)
    <variant>/base/    # standard (HA), sno (single-node), compact (schedulable CPs)
      crossplane/      # Provider IAM/identity resources
      hive/            # Namespace, ClusterDeployment, MachinePool, ManagedCluster, etc.
```

Each cluster overlay in `provision/<id>/` has `crossplane/` and `hive/` subdirectories whose `kustomization.yaml` files reference a template base and apply JSON patches.

### Cross-file coordination (critical)

When creating or modifying a cluster overlay, these values **must match** across multiple patch files:

- **Cluster name** → appears in Namespace, ClusterDeployment, MachinePool, ManagedCluster, KlusterletAddonConfig, Secret, all Crossplane resources, and install-config `metadata.name`
- **Region** → must match in ClusterDeployment `spec.platform.<provider>.region` and install-config `platform.<provider>.region`
- **Zones** → must be valid for the chosen region, set in install-config for both `controlPlane` and `compute[0]`
- **Install-config secret name** → must be `<cluster-name>-install-config` in both ClusterDeployment `installConfigSecretRef` and Secret metadata
- **IAM resource names** → cross-referenced between Crossplane resources (user/policy/attachment/key for AWS; SA/key/IAM-member for GCP; identity/role/credential for Azure)
- **Credential namespace** → must match the cluster namespace

### Tier labels

Set on ManagedCluster: `base` (baseline), `virt` (+ OpenShift Virtualization), `ai` (+ GPU Operator + OpenShift AI).

## Creating a Cluster Overlay

Preferred: use `fleet-scaffold-cluster` from the fleet repo. Manual: copy an existing overlay and replace every `cluster-placeholder` (or old cluster name) occurrence. Always validate with `kustomize build` before pushing.

## Pre-commit

Uses [gitleaks](https://github.com/gitleaks/gitleaks) v8.18.0 for secret scanning. No other linting or formatting hooks.

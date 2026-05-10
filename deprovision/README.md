# Deprovision

Adding a sentinel file to this directory triggers the deprovision pipeline via the hub's Tekton EventListener.

## Usage

```bash
touch deprovision/<cluster-id>
git add deprovision/<cluster-id>
git commit -m "deprovision: <cluster-id>"
git push
```

The deprovision pipeline deletes cluster CRs in controlled order, waits for Hive uninstall, and cleans up hub artifacts. On success, the GitHub Action archives the cluster spec:

```
git mv provision/<cluster-id> archive/<cluster-id>
rm deprovision/<cluster-id>
```

## One Cluster Per Push

The same EventListener constraint applies — push one sentinel file per commit.

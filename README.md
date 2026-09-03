# Rancher-managed RKE2 lifecycle

This AWX project snapshots and upgrades one imported, single-node RKE2 cluster
through the Rancher API. Cluster maintenance uses Rancher's downstream
Kubernetes proxy; no node SSH is required by the playbook.

## Required AWX configuration

The inventory host must define:

```yaml
rancher_url: https://rancher.example.com
clusterid: c-m-example
clustername: downstream-cluster
```

Attach the Rancher admin API key using an **OpenShift or Kubernetes API Bearer
Token** credential. It must inject:

- `K8S_AUTH_API_KEY`
- `K8S_AUTH_VERIFY_SSL`

The job survey supplies the exact target through `rke2_target_version`, for
example `v1.36.3+rke2r1`.

Use `rke2lifecycle.yaml` as the job-template playbook.

## Workflow

1. Confirm Rancher reports the cluster Ready and Connected.
2. Confirm the downstream cluster has one Ready control-plane/etcd/worker node.
3. Reject downgrades, major upgrades, and skipped minor releases.
4. Ask Rancher to create an etcd snapshot and verify the resulting downstream
   `ETCDSnapshotFile`.
5. Update the Rancher provisioning cluster to the requested RKE2 version.
6. Wait for the node, provisioning resource, and Rancher management cluster to
   converge on that version.

A repeated run at the current version is idempotent and skips both the snapshot
and upgrade request.

## Files

- `rke2lifecycle.yaml`: stable AWX entrypoint.
- `collections/ansible_collections/k8s/rke2lifecycle/roles/rke2lifecycle`:
  lifecycle implementation.
- `collections/requirements.yml` and `requirements.txt`: execution
  environment dependencies.

## Safety

The role intentionally supports exactly one all-roles node. A real version
change is never requested without a newly verified snapshot. A single-node
control plane is unavailable while RKE2 restarts, so the upgrade wait is set to
40 minutes. Keep Rancher certificate verification enabled outside lab
environments and never store API keys in source control.

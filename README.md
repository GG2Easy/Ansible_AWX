# Ansible AWX Automation

This repository contains Ansible playbooks intended to run from AWX.

## Rancher-managed RKE2 upgrades

Use `rke2_upgrade.yml` to trigger upgrades for Rancher-created RKE2 clusters.

For Rancher v2.11, use `rke2_upgrade_rancher_2_11.yml`. It performs the same workflow through Rancher's Kubernetes-style API endpoints under `/apis/...`, which are documented in the Rancher 2.11 API reference.

The playbook does **not** SSH to the RKE2 nodes and does **not** run the local RKE2 installer. It patches the Rancher provisioning cluster object by setting `spec.kubernetesVersion`. Rancher then performs the node rollout and keeps its cluster state in sync.

This matters because local node upgrades can leave Rancher stuck in states such as `waiting for kubelet to update`.

## AWX job template

Recommended AWX job template settings:

- **Playbook:** `rke2_upgrade.yml`
- **Rancher v2.11 playbook:** `rke2_upgrade_rancher_2_11.yml`
- **Inventory:** localhost inventory, or any inventory with localhost available
- **Credentials:** no SSH credential required for the target nodes
- **Rancher token:** store as an AWX credential or encrypted variable
- **Extra variables:** set Rancher URL, token, and the cluster upgrade list

Example:

```yaml
rancher_url: https://rancher.example.local
rancher_token: token-xxxxx:yyyyyyyyyyyyyyyyyyyy
rancher_validate_certs: false
rancher_create_snapshot: true
rancher_snapshot_wait: true

rke2_cluster_upgrades:
  - name: lab-single-node
    namespace: fleet-default
    version: v1.30.6+rke2r1
  - name: lab-three-node
    namespace: fleet-default
    version: v1.29.10+rke2r1
  - name: lab-six-node
    namespace: fleet-default
    version: v1.30.6+rke2r1
```

Each cluster entry can use a different desired version. The current version is read from Rancher before the upgrade is triggered.

Use the Rancher provisioning cluster object name in `name`. In many labs this matches the UI cluster name, but if it does not, use the name from the Rancher cluster URL/API object.

## Supported cluster layouts

The same playbook handles these Rancher-managed layouts because Rancher owns the rollout:

- Single-node cluster where one VM has `controlplane`, `etcd`, and `worker` roles.
- Three-node cluster where all nodes have `controlplane`, `etcd`, and `worker` roles.
- Six-node cluster with three `controlplane`/`etcd` nodes and three worker nodes.

Do not target individual RKE2 nodes with this playbook. Target Rancher cluster objects.

## Variables

- `rancher_url`: Rancher base URL, for example `https://rancher.example.local`.
- `rancher_token`: Rancher API bearer token.
- `rancher_validate_certs`: Validate Rancher TLS certificate. Defaults to `true`; set `false` for lab/self-signed certificates.
- `rancher_cluster_namespace`: Default namespace for cluster objects. Defaults to `fleet-default`.
- `rke2_cluster_upgrades`: Required list of cluster upgrade requests.
- `rancher_create_snapshot`: Request a Rancher-managed one-time etcd snapshot before changing the Kubernetes version. Defaults to `true`.
- `rancher_snapshot_wait`: Wait for Rancher to report the one-time etcd snapshot as finished before starting the upgrade. Defaults to `true`.
- `rancher_snapshot_wait_retries`: Number of snapshot status checks. Defaults to `60`.
- `rancher_snapshot_wait_delay`: Seconds between snapshot status checks. Defaults to `10`.
- `rancher_wait`: Wait for Rancher to report the cluster Ready after triggering the upgrade. Defaults to `true`.
- `rancher_wait_retries`: Number of status checks. Defaults to `120`.
- `rancher_wait_delay`: Seconds between status checks. Defaults to `30`.

If a cluster object is not in `fleet-default`, set `namespace` on that cluster entry.

## Snapshot behavior

Before changing `spec.kubernetesVersion`, the playbook requests a Rancher-managed one-time etcd snapshot by incrementing:

```yaml
spec:
  rkeConfig:
    etcdSnapshotCreate:
      generation: <next number>
```

Rancher performs the snapshot through the downstream RKE2/K3s engine. Snapshot files are stored according to the cluster's etcd snapshot configuration, either locally on etcd nodes or in the configured S3 target.

After requesting the snapshot, the playbook polls the Rancher `RKEControlPlane` object and waits for:

```yaml
status:
  etcdSnapshotCreatePhase: Finished
```

By default, the RKEControlPlane object name is assumed to match the cluster name. If it differs, set `rke_control_plane_name` on that cluster entry:

```yaml
rke2_cluster_upgrades:
  - name: rke2
    namespace: fleet-default
    rke_control_plane_name: rke2
    version: v1.30.6+rke2r1
```

## Troubleshooting

If the playbook fails while reading the Rancher cluster spec, check the sanitized HTTP status:

- `connection-error`: AWX cannot reach Rancher. From the AWX execution environment, verify `https://rancher.local/ping` resolves and returns `pong`.
- `401`: the Rancher token is invalid, expired, or copied incorrectly.
- `403`: the token is valid but does not have permission to read or update the cluster.
- `404`: the `name` or `namespace` does not match the Rancher provisioning cluster object.
- `400` or `422`: Rancher found the cluster but rejected the requested version or object update.

To confirm the cluster object name, open:

```text
https://rancher.local/v1/provisioning.cattle.io.clusters
```

Use the object `metadata.name` and `metadata.namespace` values in `rke2_cluster_upgrades`.

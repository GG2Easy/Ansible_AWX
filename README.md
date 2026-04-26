# Ansible AWX Automation

This repository contains Ansible playbooks intended to run from AWX.

## Rancher-managed RKE2 upgrades

Use `rke2_upgrade.yml` to trigger upgrades for Rancher-created RKE2 clusters.

The playbook does **not** SSH to the RKE2 nodes and does **not** run the local RKE2 installer. It patches the Rancher provisioning cluster object by setting `spec.kubernetesVersion`. Rancher then performs the node rollout and keeps its cluster state in sync.

This matters because local node upgrades can leave Rancher stuck in states such as `waiting for kubelet to update`.

## AWX job template

Recommended AWX job template settings:

- **Playbook:** `rke2_upgrade.yml`
- **Inventory:** localhost inventory, or any inventory with localhost available
- **Credentials:** no SSH credential required for the target nodes
- **Rancher token:** store as an AWX credential or encrypted variable
- **Extra variables:** set Rancher URL, token, and the cluster upgrade list

Example:

```yaml
rancher_url: https://rancher.example.local
rancher_token: token-xxxxx:yyyyyyyyyyyyyyyyyyyy
rancher_validate_certs: false

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
- `rancher_wait`: Wait for Rancher to report the cluster Ready after triggering the upgrade. Defaults to `true`.
- `rancher_wait_retries`: Number of status checks. Defaults to `120`.
- `rancher_wait_delay`: Seconds between status checks. Defaults to `30`.

If a cluster object is not in `fleet-default`, set `namespace` on that cluster entry.

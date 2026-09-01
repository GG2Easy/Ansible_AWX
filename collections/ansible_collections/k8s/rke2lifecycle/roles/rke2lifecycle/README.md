# rke2lifecycle

Safely upgrades a Rancher-managed, single-node RKE2 cluster through Rancher's
provisioning API.

The role validates cluster health and the requested release. Before a real
version change, it asks Rancher to create an etcd snapshot and waits for the
new snapshot record. It then updates the provisioning cluster version and
waits for the node, provisioning resource, and management cluster to converge.
Repeated runs at the requested version are idempotent and skip the snapshot.

## Requirements

- Inventory variables `rancher_url`, `clusterid`, and `clustername`
- A Rancher admin API key exposed through `K8S_AUTH_API_KEY`
- The `kubernetes.core` collection and Python Kubernetes client

## Main variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `rke2lifecycle_api_url` | `rancher_url` | Rancher base URL |
| `rke2lifecycle_api_key` | `K8S_AUTH_API_KEY` | Rancher bearer token |
| `rke2lifecycle_validate_certs` | `K8S_AUTH_VERIFY_SSL` or `true` | Validate TLS certificates |
| `rke2lifecycle_cluster_id` | `clusterid` | Rancher downstream cluster ID |
| `rke2lifecycle_cluster_name` | `clustername` | Rancher provisioning cluster name |
| `rke2lifecycle_target_version` | `rke2_target_version` | Exact target RKE2 tag |
| `rke2lifecycle_snapshot_timeout_seconds` | `900` | Snapshot timeout |
| `rke2lifecycle_upgrade_timeout_seconds` | `2400` | Node upgrade timeout |
| `rke2lifecycle_health_timeout_seconds` | `600` | Rancher reconciliation timeout |

## Example playbook

```yaml
---
- name: Upgrade RKE2
  hosts: all
  run_once: true
  connection: local
  gather_facts: false
  roles:
    - role: k8s.rke2lifecycle.rke2lifecycle
```

The role rejects major upgrades, downgrades, skipped minor releases, unhealthy
clusters, unexpected node topology, and upgrades without a verified snapshot.

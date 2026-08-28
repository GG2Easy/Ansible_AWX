# rke2lifecycle

Safely upgrades a Rancher-managed, single-node RKE2 cluster after creating and
verifying an etcd snapshot.

## Requirements

- A working System Upgrade Controller in the downstream cluster
- Rancher API credentials supplied through the standard `K8S_AUTH_*` variables
- The `kubernetes.core` collection and Python Kubernetes client

## Main variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `rke2lifecycle_api_url` | `K8S_AUTH_HOST` | Rancher base URL |
| `rke2lifecycle_api_key` | `K8S_AUTH_API_KEY` | Rancher bearer token |
| `rke2lifecycle_validate_certs` | `K8S_AUTH_VERIFY_SSL` or `true` | Validate TLS certificates |
| `rke2lifecycle_cluster_id` | `c-n6thr` | Rancher downstream cluster ID |
| `rke2lifecycle_target_version` | legacy `rke2_target_version` or empty | Exact target RKE2 tag |
| `rke2lifecycle_expected_node_count` | `1` | Required node count |
| `rke2lifecycle_verify_release_tag` | `true` | Check the target against GitHub releases |
| `rke2lifecycle_snapshot_timeout_seconds` | `900` | Snapshot timeout |
| `rke2lifecycle_upgrade_timeout_seconds` | `2400` | Upgrade timeout |
| `rke2lifecycle_health_timeout_seconds` | `600` | Post-upgrade health timeout |

Every configurable value is defined in `defaults/main.yml` and documented in
`meta/argument_specs.yml`.

## Example playbook

```yaml
---
- name: Upgrade RKE2
  hosts: localhost
  connection: local
  gather_facts: false
  roles:
    - role: k8s.rke2lifecycle.rke2lifecycle
      rke2lifecycle_cluster_id: c-n6thr
      rke2lifecycle_target_version: v1.35.7+rke2r1
```

## Safety behavior

The role rejects major upgrades, downgrades, skipped minor releases, unhealthy
clusters, unexpected node topology, and upgrades without a verified snapshot.

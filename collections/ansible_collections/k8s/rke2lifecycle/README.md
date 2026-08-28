# Ansible Collection - k8s.rke2lifecycle

Guarded snapshot and upgrade automation for a Rancher-managed RKE2 cluster.

## Included content

- `k8s.rke2lifecycle.rke2lifecycle`: validates the cluster and requested
  release, creates an etcd snapshot, submits a System Upgrade Controller Plan,
  and verifies post-upgrade health.

## Requirements

- `ansible-core >= 2.15`
- `kubernetes.core >= 5.1.0,<6.0.0`
- Python `kubernetes`, `PyYAML`, and `jsonpatch` packages on the execution node
- Rancher bearer token with access to the management and downstream APIs

Install the Ansible dependency from the repository root:

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

## Usage

The role reads standard Kubernetes credential variables injected by AWX:

- `K8S_AUTH_HOST`: Rancher base URL
- `K8S_AUTH_API_KEY`: Rancher bearer token
- `K8S_AUTH_VERIFY_SSL`: certificate verification setting

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

The legacy `rke2_target_version`, `rancher_url`, and `clusterid` variables are
accepted as compatibility aliases.

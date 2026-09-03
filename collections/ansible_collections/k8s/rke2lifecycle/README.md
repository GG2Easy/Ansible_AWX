# k8s.rke2lifecycle

An Ansible collection containing the `rke2lifecycle` role for snapshotting and
upgrading a single-node, Rancher-managed RKE2 cluster.

The role expects inventory variables `rancher_url`, `clusterid`, and
`clustername`, the AWX survey variable `rke2_target_version`, and a Rancher API
key injected as `K8S_AUTH_API_KEY`.

The repository root contains the AWX playbook and full operating instructions.

# Ansible AWX Automation

This repository contains Ansible playbooks intended to run from AWX.

## RKE2 upgrade

Use `rke2_upgrade.yml` to upgrade RKE2 nodes already registered in AWX inventory.

Recommended AWX job template settings:

- **Playbook:** `rke2_upgrade.yml`
- **Inventory:** the inventory that contains the RKE2 VM or RKE2 node group
- **Limit:** the RKE2 host or group you want to upgrade
- **Privilege escalation:** enabled
- **Credentials:** SSH credential with sudo access
- **Extra variables:**

```yaml
rke2_version: v1.30.6+rke2r1
target_hosts: all
rke2_drain_node: true
rke2_reboot: false
```

`rke2_version` is required. Use the exact RKE2 release version you want to install.

Optional variables:

- `target_hosts`: Ansible host pattern. Defaults to `all`; AWX inventory and limit should still target only the RKE2 VM/group.
- `rke2_drain_node`: Drain and uncordon the node when local `kubectl` access exists. Defaults to `true`.
- `rke2_reboot`: Reboot the VM after the package upgrade and service restart. Defaults to `false`.
- `rke2_node_name`: Kubernetes node name. Defaults to `ansible_hostname`; override it if the Kubernetes node name differs from the VM hostname.
- `rke2_wait_timeout`: Seconds to wait for reboot and Kubernetes Ready checks. Defaults to `600`.

The playbook upgrades one node at a time (`serial: 1`). For `rke2-server` nodes it creates an etcd snapshot before changing the installed version.

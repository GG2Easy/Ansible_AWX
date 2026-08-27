# Rancher-managed RKE2 upgrade with AWX

This project performs a guarded upgrade of the imported, single-node RKE2
cluster registered in Rancher. It uses one Rancher bearer token stored in the
requested built-in AWX credential and a job-injecting credential. It does not
use SSH for cluster maintenance and does not use `kdeploy`.

The workflow:

1. Authenticates to Rancher with the AWX-injected bearer token.
2. Confirms Rancher reports the downstream cluster active and non-transitioning.
3. Confirms the node is Ready and has control-plane, etcd, and worker roles.
4. Validates the requested RKE2 release and rejects downgrades or skipped
   minor versions.
5. Creates an `ETCDSnapshotSave` operation in Rancher.
6. Waits for the operation to reach `Succeeded`.
7. Confirms the corresponding downstream `ETCDSnapshotFile` exists.
8. Creates or updates an RKE2 `Plan` for the existing System Upgrade
   Controller.
9. Waits for the Plan to complete, then confirms the node version, node
   readiness, and Rancher cluster health.

## Repository layout

```text
.
|-- awx/survey_spec.json
|-- collections/requirements.yml
|-- execution-environment.yml
|-- inventory/localhost.yml
|-- playbooks/rke2_upgrade.yml
|-- requirements.txt
```

## AWX credential

Create a credential using the built-in type
**OpenShift or Kubernetes API Bearer Token**:

- **Name:** `Rancher RKE2 Bearer Token`
- **OpenShift or Kubernetes API Endpoint:** `https://192.168.56.2`
- **API authentication bearer token:** the Rancher automation token
- **Verify SSL:** disabled only for this lab's self-signed Rancher endpoint

AWX reserves that built-in type for Kubernetes container groups; it has no
job-template injectors. Create a second credential type named
`Rancher Kubernetes API Bearer Token - Job` with these environment
injectors, then create `Rancher RKE2 Bearer Token - Job` from the same
endpoint and token:

```yaml
env:
  K8S_AUTH_HOST: "{{ host }}"
  K8S_AUTH_API_KEY: "{{ bearer_token }}"
  K8S_AUTH_VERIFY_SSL: "{{ verify_ssl }}"
```

Attach both credentials to the job template. This keeps the requested built-in
**OpenShift or Kubernetes API Bearer Token** credential visible on the
template, while the custom credential supplies the environment injectors that
the managed built-in type does not provide. The supplied VM configuration
script creates and attaches both credentials. The role uses the Rancher API
for the snapshot operation and the Rancher downstream proxy at
`/k8s/clusters/c-n6thr` for Kubernetes resources.

For production, enable certificate verification and place Rancher's CA in the
credential. Rotate the token before its expiry and immediately revoke it if it
is exposed.

## AWX project and job template

Create a Git project from:

```text
https://github.com/GG2Easy/Ansible_AWX.git
```

Create a job template with:

- **Playbook:** `playbooks/rke2_upgrade.yml`
- **Inventory:** a localhost inventory, or the existing `RKE2` inventory
- **Credentials:** `Rancher RKE2 Bearer Token` and
  `Rancher RKE2 Bearer Token - Job`
- **Survey enabled:** yes

Import [awx/survey_spec.json](awx/survey_spec.json) as the survey definition.
The required survey variable is `rke2_target_version`. It is a single-select
multiple-choice question with these currently offered upgrade targets:

- `v1.35.6+rke2r1`
- `v1.35.7+rke2r1` (default)

Both are one minor release above the lab's initial `v1.34.10+rke2r1`. Update
the choices as the environment changes. The role still verifies that the exact
tag exists in the official RKE2 GitHub releases.

## Local syntax check

```bash
ansible-galaxy collection install -r collections/requirements.yml
ansible-playbook --syntax-check playbooks/rke2_upgrade.yml
```

To run outside AWX, export the same variables injected by the credential:

```bash
export K8S_AUTH_HOST=https://rancher.example.com
export K8S_AUTH_API_KEY='token-id:secret'
export K8S_AUTH_VERIFY_SSL=true
ansible-playbook playbooks/rke2_upgrade.yml -e rke2_target_version=v1.35.7+rke2r1
```

Do not store the token in inventory, source control, job extra variables, or
the survey.

## Safety constraints

- The role is intentionally configured for one all-roles RKE2 node.
- A single-node control plane has no high availability; the Kubernetes API is
  temporarily unavailable while RKE2 restarts.
- Only forward upgrades are accepted.
- A minor release may advance by at most one at a time.
- The upgrade is not submitted until the snapshot operation succeeds and the
  downstream snapshot record is visible.
- The upgrade Plan remains in the cluster as the declared desired version.
- Local snapshots protect against upgrade mistakes but not loss of the VM.
  Production clusters should copy snapshots to S3-compatible storage.

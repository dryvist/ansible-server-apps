# openbao_lxc

Deploys self-hosted OpenBao in a privileged LXC via Docker Compose.
Manual unseal initially; auto-unseal via age-encrypted Shamir shares
stored in SOPS planned for follow-up.

This role is a skeleton today (`tasks/main.yml` only emits a debug
notice). Implementation lands in a follow-up PR.

## Installation

```bash
ansible-galaxy collection install -r requirements.yml
```

This role lives inside the repo and is consumed via `playbooks/site.yml`
once enabled. No standalone install step.

## Usage

```bash
sops exec-env secrets.enc.yaml \
  'ansible-playbook -i inventory/hosts.yml playbooks/site.yml \
    --tags openbao_lxc'
```

Configure via `roles/openbao_lxc/defaults/main.yml` (LXC ID,
datastore, unseal-key path, admin email, image pin, API + cluster
ports).

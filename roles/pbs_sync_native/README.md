# pbs_sync_native

Configures Proxmox Backup Server sync via the native
`proxmox-backup-manager` CLI. The `community.proxmox` PBS modules
(`proxmox_backup_remote`, `proxmox_backup_sync`) are vaporware in
2026, so this role wraps the CLI via `ansible.builtin.command` with
explicit `changed_when` guards driven by JSON parsing of
`--output-format json`.

This role is a skeleton today (`tasks/main.yml` only emits a debug
notice). Implementation lands in a follow-up PR.

## Installation

```bash
ansible-galaxy collection install -r requirements.yml
```

This role is consumed inside the repo via `playbooks/site.yml`. No
standalone install step.

## Usage

```bash
sops exec-env secrets.enc.yaml \
  'ansible-playbook -i inventory/hosts.yml playbooks/site.yml \
    --tags pbs_sync_native'
```

Configure via `roles/pbs_sync_native/defaults/main.yml`
(`pbs_sync_native_remotes` + `pbs_sync_native_jobs`). Today both
default to empty lists, so the role is a no-op until the cluster +
PBS LXCs exist.

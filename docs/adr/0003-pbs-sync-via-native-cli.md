# 0003 -- PBS Sync via the native proxmox-backup-manager CLI

Status: Accepted (2026-05-10)

## Context

Proxmox Backup Server (PBS) is the backup target for the cluster.
Each cluster node runs a PBS LXC, and the three instances form a
sync ring (`B -> C -> D -> B`) so that any single-node loss is
recoverable from another node's PBS datastore.

We need Ansible to manage:

- PBS remotes (`proxmox-backup-manager remote create`)
- PBS sync jobs (`proxmox-backup-manager sync-job create`)

Considered:

1. **`community.proxmox.proxmox_backup_remote` /
   `proxmox_backup_sync`** -- modules with these names exist as
   tracking issues in the `community.proxmox` repo, but as of 2026
   they are vaporware; no code, no PR, no roadmap commitment.
2. **`ansible.builtin.shell` blocks** -- works, but no idempotence
   without bespoke logic, and "shell with logic" is
   structurally a script in disguise (banned by the no-scripts rule).
3. **`ansible.builtin.command` + native `proxmox-backup-manager`
   CLI + `--output-format json` + explicit `changed_when` guards
   driven by JSON parsing** -- idempotence is structural; the CLI
   is the supported, vendor-blessed automation surface.

## Decision

Wrap `proxmox-backup-manager` via `ansible.builtin.command` with
explicit `changed_when` guards based on JSON parsing of
`--output-format json` output. The `pbs_sync_native` role owns this
wiring. Inputs are `pbs_sync_native_remotes` (list) and
`pbs_sync_native_jobs` (list), shaped to match
`proxmox-backup-manager`'s native CLI.

## Consequences

- Idempotence is real: we list current state, compute the diff, only
  call `create`/`update` for items that actually need it.
- When `community.proxmox` ships real PBS modules, we swap the
  internals and keep the role's input contract.
- The role is currently a skeleton; the implementation PR depends on
  the cluster + PBS LXCs existing.

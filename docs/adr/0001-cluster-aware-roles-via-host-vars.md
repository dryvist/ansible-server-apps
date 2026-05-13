# 0001 -- Cluster-aware roles via host_vars (not duplicated playbooks)

Status: Accepted (2026-05-10)

## Context

This repo manages app deployments on a Proxmox cluster (today: 1
node `pve`; tomorrow: 3-node cluster `proxmox-{b,c,d}`). The
inherited single-host shape ("everything on `pve`") doesn't survive
the cluster move. We need a way to say "Cribl Edge runs on each
cluster node, but its persistent queue dataset lives on the local
ZFS pool" without copy-pasting plays.

## Decision

Roles stay node-agnostic. Per-node specifics live in `host_vars/`,
keyed by inventory hostname. The dynamic inventory loader
(`inventory/load_terraform.yml`) consumes the
`dryvist/homelab-schemas` `ansible-inventory-v1.json` shape, which
supplies `cluster`, `hosts.*`, `ha_groups`, `replication_jobs`, and
`pbs_targets` blocks. Roles read these via `terraform_data.*`
(variable name preserved for backwards-compat with the 17 inherited
roles).

`playbooks/site.yml` sets `serial: 1` as the global default for plays
that target the cluster (apps deployed across all 3 nodes), so
node-by-node rollout is the default rather than a special case.
Per-role overrides are allowed via play-level `serial:` keyword.

## Consequences

- Adding the 4th cluster node is one inventory entry plus one
  `host_vars/proxmox-d.yml`, not a playbook fork.
- We pay one inventory contract: the schema is owned by
  `dryvist/homelab-schemas` and pinned via `requirements.yml`. Any
  schema break shows up in CI before it shows up in production.
- `serial: 1` adds wall-time. Roles that genuinely need parallel
  rollout (idempotent, no shared mutable state) opt out explicitly.

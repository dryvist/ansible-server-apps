# 0004 -- Splunk linkage via docs only

Status: Accepted (2026-05-10)

## Context

Splunk Enterprise is the downstream HEC consumer for Cribl Edge and
Cribl Stream output. The original `JacobPEvans/ansible-proxmox-apps`
repo carried `playbooks/splunk-docker.yml` and treated Splunk as a
peer app it deployed. As part of the dryvist split, Splunk is no
longer co-owned with the Cribl side -- it stays at
`JacobPEvans/ansible-splunk` and is referenced by docs only.

Considered:

1. **Submodule** -- pulls Splunk source into this repo; tightens
   coupling we are explicitly trying to loosen.
2. **`requirements.yml` git source** -- works for Ansible roles, but
   `ansible-splunk` is a playbook tree, not a role package.
3. **Inventory-only linkage with docs that point at the upstream
   repo** -- loosest possible coupling; Splunk can move, fork, or
   restructure without breaking anything here.

## Decision

This repo references Splunk only through the inventory's `splunk_vm`
host group (so Cribl knows where to push HEC events). It carries no
Splunk roles, no Splunk playbooks, and no Splunk-side validation.

`playbooks/validate-pipeline.yml` validates the Cribl side only:
HAProxy config, Cribl Edge / Stream service health, NetFlow port
listener. The "did the event land in the Splunk index" check is
owned by `JacobPEvans/ansible-splunk`'s own validation tree.

## Consequences

- Splunk migrations (e.g., the parallel-HEC cutover -- see ADR
  0010) are coordinated via cross-repo issues, not Ansible role
  changes.
- The original `splunk_vm` group definition stays in
  `inventory/load_terraform.yml` -- it's just an HEC destination,
  not a deployment target.
- If Splunk ever moves to dryvist (currently: no -- per the day-zero
  plan, Q23), we revisit this ADR.

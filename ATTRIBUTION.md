# Attribution

`dryvist/ansible-server-apps` was forked from
[`JacobPEvans/ansible-proxmox-apps`](https://github.com/JacobPEvans/ansible-proxmox-apps)
via a full mirror push (history preserved) on 2026-05-10.

## Source

- Upstream: <https://github.com/JacobPEvans/ansible-proxmox-apps>
- Original author: Jacob P. Evans (<https://github.com/JacobPEvans>)
- License at fork: Apache-2.0
- License going forward (this fork): MIT

The MIT license was chosen for the dryvist organization. Apache-2.0
permits relicensing under MIT. The original copyright is preserved in
the new `LICENSE` file alongside the dryvist copyright.

## What changed at fork

- 17 inherited roles (cribl_edge, cribl_stream, cribl_packs,
  cribl_docker_stack, haproxy, technitium_dns, technitium_install,
  apt_cacher_ng, mailpit_docker, ntfy_docker, mssql_docker,
  qdrant_docker, llamaindex, github_runner, minio,
  systemd_restart_policy, plus pipeline glue) carried over verbatim.
- Splunk-specific items stripped: `playbooks/splunk-docker.yml` does
  not exist in this repo; `playbooks/validate-pipeline.yml` validates
  only the Cribl-to-HEC side. Splunk Enterprise stays at
  [`JacobPEvans/ansible-splunk`](https://github.com/JacobPEvans/ansible-splunk)
  and is referenced by docs only — see
  `docs/adr/0004-splunk-linkage-via-docs-only.md` and
  `docs/adr/0010-splunk-parallel-hec-cutover.md`.
- 3 new role skeletons: `openbao_lxc`, `openbao_agent`,
  `pbs_sync_native` (implementation in follow-up PRs).
- Inventory file renamed: `terraform_inventory.json` ->
  `ansible_inventory.json`. Schema sourced from
  [`dryvist/homelab-schemas`](https://github.com/dryvist/homelab-schemas).
- Doppler removed from runtime secrets handling. SOPS+age is the
  bootstrap path; OpenBao becomes the runtime path once
  `roles/openbao_lxc` is implemented. Doppler is retained ONLY for the
  `gh-workflow-tokens` project (PAT injection for runner registration).
- License: MIT (was Apache-2.0).
- Cluster-aware execution: `serial: 1` global default for plays that
  target the (future) 3-node Proxmox cluster.
- Drift detection: weekly `ansible-playbook --check --diff` cron.

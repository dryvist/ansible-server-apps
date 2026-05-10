# AI Agents Configuration

@JacobPEvans/ai-assistant-instructions

## Repo Scope

Ansible application deployments inside the dryvist Proxmox cluster
(VMs and LXC containers). VM/LXC provisioning lives in
`dryvist/tofu-proxmox-cluster`; host-level Proxmox config lives in
`dryvist/ansible-proxmox-cluster`; this repo handles app-level config
only.

Splunk is NOT owned here — it stays at
`JacobPEvans/ansible-splunk` and is linked via docs only (see
`docs/adr/0004-splunk-linkage-via-docs-only.md`).

## Locked decisions for this repo

- 17 roles inherited verbatim from `JacobPEvans/ansible-proxmox-apps`.
- 3 new roles, skeletons today, implementation in follow-up PRs:
  `openbao_lxc`, `openbao_agent`, `pbs_sync_native`.
- Inventory shape sourced from `dryvist/homelab-schemas` via
  `requirements.yml` git source.
- `serial: 1` global default in cluster-targeted plays
  (`playbooks/site.yml`); per-role overrides allowed.
- Secrets bridge variables: `<role>_secret_source` +
  `openbao_available`. Default `secret_source: sops` until OpenBao is
  reachable; flip per-role one PR at a time.
- Doppler retained ONLY for `gh-workflow-tokens` project (PAT
  injection for `github_runner` role); dropped from all other runtime
  flows.
- Drift detection: weekly `ansible-playbook --check --diff` cron in
  `.github/workflows/drift-check-cron.yml`, alerts to ntfy.
- Mermaid diagrams are committed as `.mmd` source AND rendered `.svg`
  per `JacobPEvans/.github` PR #620; CI gate enforces re-render.

## How to run things

See `CLAUDE.md` for command snippets and tooling notes.

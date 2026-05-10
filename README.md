# dryvist/ansible-server-apps

Ansible application deployments inside the dryvist Proxmox cluster
(VMs and LXC containers).

VM and LXC provisioning lives in
[`dryvist/tofu-proxmox-cluster`](https://github.com/dryvist/tofu-proxmox-cluster).
Host-level Proxmox config (cluster init, ZFS, PBS LXC) lives in
[`dryvist/ansible-proxmox-cluster`](https://github.com/dryvist/ansible-proxmox-cluster).
Splunk Enterprise is owned by
[`JacobPEvans/ansible-splunk`](https://github.com/JacobPEvans/ansible-splunk)
and linked from this repo via docs only.

This repo was forked from
[`JacobPEvans/ansible-proxmox-apps`](https://github.com/JacobPEvans/ansible-proxmox-apps)
on 2026-05-10. See [`ATTRIBUTION.md`](ATTRIBUTION.md).

## Ecosystem context

```text
dryvist/tofu-proxmox-cluster   <- provisions VMs + LXCs, emits ansible_inventory.json
                |
                v
   dryvist/ansible-server-apps  <- THIS REPO: deploys apps inside those VMs/LXCs
                |
                v
        JacobPEvans/ansible-splunk  <- Splunk Enterprise (downstream HEC consumer)
```

Full diagram: [`docs/assets/ecosystem-context.svg`](docs/assets/ecosystem-context.svg)

## What this repo deploys

- **Cribl Edge** -- native install on LXC, syslog ingestion
- **Cribl Stream** -- native install on LXC, NetFlow/IPFIX ingestion
- **Cribl Packs** -- pack assets on Cribl LXCs
- **HAProxy** -- syslog/netflow VIP forwarding to Cribl LXCs
- **Technitium DNS** -- internal DNS server
- **apt-cacher-ng** -- APT package proxy
- **Mailpit** -- SMTP relay + web UI
- **ntfy** -- push notification server
- **MinIO** -- S3-compatible object storage
- **MSSQL** -- SQL Server in Docker on LXC
- **GitHub Actions Runners** -- Docker Compose on docker-host VM
- **Qdrant** -- vector database (Docker in LXC)
- **LlamaIndex + Ollama** -- CPU-only embeddings on LXC
- **systemd_restart_policy** -- cross-cutting hardening role

### Skeleton roles (today) -- implementation in follow-up PRs

- **openbao_lxc** -- self-hosted OpenBao in privileged LXC (Docker
  Compose; vendor-only deployment)
- **openbao_agent** -- vault-agent on each consumer LXC, renders
  `EnvironmentFile` snippets for systemd services
- **pbs_sync_native** -- Proxmox Backup Server sync via native
  `proxmox-backup-manager` CLI

## Installation

```bash
git clone git@github.com:dryvist/ansible-server-apps.git
cd ansible-server-apps
direnv allow
ansible-galaxy collection install -r requirements.yml
```

The Nix dev shell (sourced from
`github:JacobPEvans/nix-devenv#ansible-apps` via `.envrc`) provides
`ansible`, `ansible-lint`, `molecule`, `sops`, `age`, `python3` with
the required Python deps, `jq`, `yq`, and `pre-commit`. There is no
local `flake.nix` -- direnv fetches and caches the remote shell.

Required environment material is loaded via `sops exec-env` from
`secrets.enc.yaml` (template in `secrets.enc.yaml.example`).

## Usage

```bash
# Sync inventory from the upstream OpenTofu output
# (Manual today; auto-sync planned via dryvist/tofu-proxmox-cluster output hook)

# Deploy all apps (SOPS-bridged secrets)
sops exec-env secrets.enc.yaml \
  'ansible-playbook -i inventory/hosts.yml playbooks/site.yml'

# Deploy GitHub runners (Doppler exception for runner PAT)
doppler run -p gh-workflow-tokens -c prd -- \
  sops exec-env secrets.enc.yaml \
  'ansible-playbook -i inventory/hosts.yml playbooks/site.yml \
    --tags github_runner'

# Validate Cribl-side pipeline (Splunk validation lives in ansible-splunk)
sops exec-env secrets.enc.yaml \
  'ansible-playbook -i inventory/hosts.yml playbooks/validate-pipeline.yml'

# Lint
ansible-lint
```

## Secrets

SOPS+age at rest, decrypted via `sops exec-env` for runtime today.
After the `openbao_lxc` and `openbao_agent` roles ship, runtime
secret material moves to OpenBao via `vault-agent`. Per-role
flip is gated by the `<role>_secret_source` + `openbao_available`
bridge variables (default `secret_source: sops`).

Doppler is retained ONLY for the `gh-workflow-tokens` project (runner
PAT injection); see `CLAUDE.md`.

## Documentation

- [`CLAUDE.md`](CLAUDE.md) -- AI agent + operator reference
- [`docs/adr/`](docs/adr/) -- architecture decision records
- [`docs/assets/`](docs/assets/) -- Mermaid diagrams (`.mmd` + `.svg`)
- [`ATTRIBUTION.md`](ATTRIBUTION.md) -- fork provenance

## Dev environment

Nix flake + direnv. Shell sourced from
[`JacobPEvans/nix-devenv#ansible-apps`](https://github.com/JacobPEvans/nix-devenv).
No local `flake.nix`.

```bash
direnv allow      # one-time per worktree
ansible --version # provided by the dev shell
```

## License

[MIT](LICENSE) (relicensed from the original Apache-2.0 fork).

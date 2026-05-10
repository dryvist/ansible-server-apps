# Ansible Server Apps (dryvist)

@AGENTS.md

Configure applications on Proxmox cluster VMs and LXC containers.
VMs/containers are provisioned by `dryvist/tofu-proxmox-cluster`;
this repo handles app config only. Splunk is owned by
`JacobPEvans/ansible-splunk` and linked via docs only.

## This Repo Owns

- **Cribl Edge** (`cribl_edge` role -- native install on LXC containers)
- **Cribl Stream** (`cribl_stream` role -- native install on LXC containers)
- **Cribl Packs** (`cribl_packs` role -- pack assets on Cribl LXCs)
- **HAProxy** (LXC container, syslog/netflow VIP forwarding to Cribl LXCs)
- **Technitium DNS** (LXC container)
- **apt-cacher-ng** (LXC container)
- **Mailpit** (LXC container, SMTP relay with web UI)
- **ntfy** (LXC container, push notification server)
- **MinIO** (LXC container, S3-compatible object storage)
- **MSSQL** (`mssql_docker` role -- SQL Server in Docker on LXC)
- **GitHub Actions Runners** (`github_runner` role -- Docker Compose on docker-host VM)
- **Qdrant** (`qdrant_docker` role -- Docker in LXC container)
- **LlamaIndex** (`llamaindex` role -- Python + Ollama CPU-only embeddings on LXC)
- **systemd_restart_policy** (cross-cutting role)

### Skeleton roles (implementation in follow-ups)

- **openbao_lxc** -- self-hosted OpenBao in privileged LXC via Docker Compose
- **openbao_agent** -- vault-agent on each consumer LXC (uses
  `community.hashi_vault`; OpenBao is Vault-API compatible)
- **pbs_sync_native** -- PBS Sync via native `proxmox-backup-manager`
  CLI (community.proxmox PBS modules are vaporware)

**This repo does NOT own Splunk.** Splunk migration uses parallel HEC
(see `docs/adr/0010-splunk-parallel-hec-cutover.md`).

## Pipeline Data Flow

```text
Source -> HAProxy LXC (TCP+UDP syslog + IPFIX 2055)
           |
       Cribl Edge LXCs [syslog ports]
         - Pipeline: sets index + sourcetype by port
         - Output: Splunk HEC (https)
           |
       Cribl Stream LXCs [IPFIX 2055]
         - Pipeline: sets index=network, sourcetype=ipfix
         - Output: Splunk HEC (https)
           |
       Splunk Enterprise (managed by JacobPEvans/ansible-splunk)
```

## Cluster-aware Execution

`playbooks/site.yml` sets `serial: 1` as the global default for plays
that target the cluster (apps that exist as singletons on each node,
or that need ordered upgrades). Per-role overrides are allowed via
the role's `meta/main.yml`.

## Inventory

Inventory is loaded dynamically from `inventory/ansible_inventory.json`
(produced by `dryvist/tofu-proxmox-cluster`) via
`inventory/load_terraform.yml`. The schema is pinned to
`dryvist/homelab-schemas` through `requirements.yml`.

### Schema source-of-truth

- Repo: <https://github.com/dryvist/homelab-schemas>
- File: `schemas/ansible-inventory-v1.json`
- Pinned via `requirements.yml` git source (currently `main`; will
  flip to a tagged version when v1.0.0 ships).

### Groups (from inventory)

- `lxc_containers`: All LXC containers (`proxmox_pct_remote` connection)
- `cribl_edge`: Cribl Edge LXC containers (syslog processing)
- `cribl_stream_group`: Cribl Stream LXC containers (netflow/IPFIX)
- `docker_vms` / `cribl_docker_group`: Docker hosts (testing/dev + CI)
- `mailpit_group`: Containers tagged `smtp`
- `ntfy_group`: Containers tagged `push`
- `qdrant_group`: Containers tagged `vectordb`
- `llamaindex_group`: Containers tagged `rag`
- `apt_cacher_group`: Containers tagged `apt-cache`
- `minio_group`: Containers tagged `minio`
- `technitium_dns_group`: Containers tagged `dns`
- `mssql_group`: Containers tagged `database`

## Secrets Management

**At-rest encryption**: SOPS + age (`secrets.enc.yaml`). Age key is
backed up to Bitwarden vault.

**Runtime injection (today)**: `sops exec-env` exports decrypted
values as environment variables before running ansible-playbook.

**Runtime injection (post-OpenBao)**: `vault-agent` (the
`openbao_agent` role) renders `EnvironmentFile`-style snippets per
consumer service. Roles flip from SOPS to OpenBao one PR at a time
via the `<role>_secret_source` + `openbao_available` bridge variable
pattern (default `secret_source: sops` until OpenBao is up).

See `docs/adr/0007-secrets-sops-then-openbao.md` and
`docs/adr/0008-bridge-variable-pattern.md`.

### Doppler scope (intentional carve-out)

Doppler is retained ONLY for the `gh-workflow-tokens` project, which
provides `GH_PAT_RUNNER_TOKEN` for the `github_runner` role. All
other runtime secrets flow through SOPS today and OpenBao tomorrow.

```bash
# github_runner is the ONE doppler-bridged role
doppler run -p gh-workflow-tokens -c prd -- \
  sops exec-env secrets.enc.yaml \
  'ansible-playbook -i inventory/hosts.yml playbooks/site.yml \
    --tags github_runner'
```

### Required environment variables (sourced from SOPS)

| Variable | Purpose |
| --- | --- |
| `PROXMOX_VE_HOSTNAME` | Proxmox VE hostname |
| `PROXMOX_VE_NODE` | Proxmox node name |
| `PROXMOX_VE_GATEWAY` | Network gateway (for IP derivation) |
| `PROXMOX_DOMAIN` | Internal DNS domain |
| `PROXMOX_SSH_KEY_PATH` | SSH key for Proxmox host and non-Docker VMs |
| `PROXMOX_DKR_SSH_KEY_PATH` | SSH key for Docker VM direct access |
| `SPLUNK_HEC_TOKEN` | Splunk HEC token (for Cribl output) |
| `SPLUNK_PASSWORD` | Splunk admin password (for E2E validation) |
| `HAPROXY_STATS_PASSWORD` | HAProxy stats page password |
| `TECHNITIUM_DNS_API_TOKEN` | Technitium DNS API token |
| `MAILPIT_RELAY_HOST` | SMTP relay hostname |
| `MAILPIT_RELAY_PORT` | SMTP relay port |
| `MAILPIT_RELAY_USERNAME` | SMTP relay username |
| `MAILPIT_RELAY_PASSWORD` | SMTP relay password |
| `MSSQL_SA_PASSWORD` | SQL Server SA password |
| `QDRANT_API_KEY` | Qdrant vector database API key |
| `GH_PAT_RUNNER_TOKEN` | Runner registration PAT (Doppler exception) |

## Commands

```bash
# Deploy all apps (SOPS-bridged secrets)
sops exec-env secrets.enc.yaml \
  'ansible-playbook -i inventory/hosts.yml playbooks/site.yml'

# Deploy GitHub runners (Doppler exception for runner PAT)
doppler run -p gh-workflow-tokens -c prd -- \
  sops exec-env secrets.enc.yaml \
  'ansible-playbook -i inventory/hosts.yml playbooks/site.yml \
    --tags github_runner'

# Edit encrypted secrets
sops secrets.enc.yaml

# Validate pipeline (Cribl side only; Splunk validation lives in ansible-splunk)
sops exec-env secrets.enc.yaml \
  'ansible-playbook -i inventory/hosts.yml playbooks/validate-pipeline.yml'

# Lint
ansible-lint
```

## Testing

### Fast (CI + pre-commit)

| Check | Command |
| --- | --- |
| Ansible lint | `ansible-lint` |
| Playbook syntax | `ansible-playbook --syntax-check` |
| Inventory schema validation | via `ansible.utils.validate` |
| Molecule syntax | `molecule syntax` |

```bash
cp tests/inventory_load/ansible_inventory.json inventory/ansible_inventory.json
ansible-playbook tests/inventory_load/verify_inventory.yml \
  -i inventory/hosts.yml -c local
```

### Extended (manual)

```bash
ansible-galaxy collection install -r requirements.yml
molecule test
```

## Dev Environment

Nix flakes + direnv:

```sh
direnv allow    # one-time per worktree
```

Shell is provided by the `ansible-apps` shell in
[`JacobPEvans/nix-devenv`](https://github.com/JacobPEvans/nix-devenv)
via `.envrc`. There is no local `flake.nix`.

To activate manually:

```sh
nix develop "github:JacobPEvans/nix-devenv#ansible-apps"
```

## Related Repositories

| Repo | Relationship |
| --- | --- |
| `dryvist/tofu-proxmox-cluster` | Upstream: provisions VMs/LXCs |
| `dryvist/ansible-proxmox-cluster` | Peer: Proxmox host-level config |
| `dryvist/homelab-schemas` | Schema source-of-truth (inventory contract) |
| `JacobPEvans/ansible-splunk` | Splunk Enterprise (linked via docs only) |

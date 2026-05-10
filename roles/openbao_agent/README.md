# openbao_agent

Configures vault-agent on each consumer LXC, fetching secrets from
OpenBao and rendering EnvironmentFile snippets for systemd services.

OpenBao is Vault-API compatible, so this role uses the
`community.hashi_vault` collection for direct API interactions and
the upstream `openbao` (or Vault) binary for the agent itself.

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
    --tags openbao_agent'
```

Configure via `roles/openbao_agent/defaults/main.yml` (server
endpoint, role-id / secret-id paths, env-file output dir, template
backoff). Once configured, individual consumer roles flip to OpenBao
one PR at a time via `<role>_secret_source: openbao`.

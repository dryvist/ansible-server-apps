# 0002 -- OpenBao (self-hosted), not Infisical or HashiCorp Vault OSS

Status: Accepted (2026-05-10)

## Context

We need a runtime secret store for Cribl tokens, Splunk HEC tokens,
HAProxy stats password, Mailpit relay creds, MSSQL SA password,
Qdrant API key, etc. Today these come from SOPS-decrypted env vars
via `sops exec-env`, which works fine for human-driven runs but
doesn't fit machine identities (vault-agent on each consumer LXC,
short-lived AppRole tokens, automatic rotation).

Considered:

1. **HashiCorp Vault OSS** -- license uncertainty post-IBM, requires
   a strict reading of the BSL terms for a homelab use case.
2. **Infisical** -- excellent UX, but a SaaS-first design; the
   self-hosted offering is the secondary path and the API-token
   model leaks a long-lived token onto every consumer.
3. **OpenBao** -- LF fork of Vault OSS, fully open-source license,
   Vault-API compatible (existing `community.hashi_vault` collection
   works unmodified, existing `vault-agent` binary works unmodified).
4. **Just keep SOPS** -- works for humans; doesn't fit
   machine-driven secret rotation; no audit trail.

## Decision

Self-host OpenBao in a privileged LXC (vendor-only Docker Compose
deployment, hence the privileged + nesting + keyctl + fuse flags).
Consumer LXCs run `vault-agent` (the `openbao_agent` role) which
authenticates via AppRole and renders `EnvironmentFile`-style
snippets that systemd units load via `EnvironmentFile=`.

SOPS stays for bootstrap (the OpenBao root token / unseal shares are
SOPS-encrypted in the repo) and as the fallback path during the
SOPS-to-OpenBao migration. The bridge variable
`<role>_secret_source: sops|openbao` lets each consumer role flip
independently, one PR at a time. Default `secret_source: sops` until
OpenBao is up.

## Consequences

- One privileged LXC; we accept the blast-radius tradeoff.
- We use the existing `community.hashi_vault` collection -- no new
  Ansible module to write or maintain.
- HA OpenBao (Raft) is deferred until a second cluster node is up;
  single-node OpenBao is acceptable for a homelab.
- Auto-unseal via age-encrypted Shamir shares stored in SOPS is a
  follow-up; the first cut requires a manual unseal on cold boot.

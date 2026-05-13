# 0007 -- Secrets: SOPS today, OpenBao tomorrow (gradual cutover)

Status: Accepted (2026-05-10)

## Context

We have 14+ env vars threaded through roles today (Splunk HEC token,
HAProxy stats password, Mailpit relay creds, MSSQL SA password,
etc.). They all flow through `sops exec-env` in the inherited
implementation. We've decided to stand up OpenBao (see ADR 0002),
but flipping all 14 roles in one PR is a giant unsafe blast radius.

## Decision

Two-axis cutover, governed by `inventory/group_vars/all.yml`:

1. **`openbao_available: bool`** -- global flag. Flips to `true`
   once `roles/openbao_lxc` has been deployed and
   `roles/openbao_agent` is reachable on consumer LXCs.
2. **`<role>_secret_source: sops|openbao`** -- per-role override.
   Default is `secret_source: sops`. Each consumer role checks
   `<role>_secret_source | default(secret_source)` inside its tasks
   and either reads `lookup('env', '<VAR>')` (sops path) or reads
   from `community.hashi_vault.vault_kv2_get` against
   `127.0.0.1:8200` (openbao path).

This means: until OpenBao is up, nothing changes. When OpenBao
lands, the global flag flips to `true` but every role keeps reading
from SOPS. Then each role gets its own PR that flips
`<role>_secret_source: openbao`, with rollback being a one-line
revert.

See also ADR 0008 for the bridge variable pattern in detail.

## Consequences

- Migration is reversible at the role level.
- We pay a lookup cost per task that uses `community.hashi_vault`,
  but vault-agent's templated `EnvironmentFile=` snippets keep the
  hot path fast.
- We accept that some roles may live on SOPS forever (e.g.,
  bootstrap roles that need to be runnable before OpenBao exists).

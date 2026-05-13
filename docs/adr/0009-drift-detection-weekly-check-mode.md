# 0009 -- Drift detection via weekly `ansible-playbook --check --diff`

Status: Accepted (2026-05-10)

## Context

Deploys are intentionally infrequent (this is a homelab on a 2-5
hours/week budget). Between deploys, the world drifts: someone
edits a file on a host, a container restarts with new env vars, an
upstream package update changes a default. Without a periodic
check, we discover drift only when something breaks.

Considered:

1. **A separate enforcement run on a cron** -- enforces, but also
   masks intentional out-of-band changes (e.g., a temporary debug
   tweak) by silently reverting them.
2. **A periodic `ansible-playbook --check --diff` run with no
   enforcement** -- detects drift, surfaces it as a notification,
   leaves the decision to the operator.
3. **No drift detection** -- we discover drift via incidents.

## Decision

A weekly GitHub Actions cron runs
`ansible-playbook --check --diff -i inventory/hosts.yml playbooks/site.yml`
against the cluster, captures the diff, and posts a summary to ntfy
via the `community.general.ntfy` action (or curl, if the action
isn't available in the runner). No automatic enforcement.

Workflow lives at `.github/workflows/drift-check-cron.yml`.

## Consequences

- Drift becomes a chat notification, not an outage.
- The check workflow needs SOPS-decrypted env vars (HEC tokens,
  etc.) to run -- it pulls them from a GitHub Actions secret holding
  the decrypt-key material.
- Some roles intentionally produce diff in `--check` mode (e.g.,
  ones that template based on facts that change every run). We
  audit those one-by-one; whitelisting a noisy role is acceptable.

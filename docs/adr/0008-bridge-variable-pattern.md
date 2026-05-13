# 0008 -- Bridge variable pattern: `<role>_secret_source` + `openbao_available`

Status: Accepted (2026-05-10)

## Context

ADR 0007 commits us to a per-role SOPS-to-OpenBao cutover. Without
a clear convention, every role would invent its own toggle name,
defaulting condition, and fallback behavior. That fragments the
reader's mental model and makes bulk audits ("which roles have
already moved to OpenBao?") expensive.

## Decision

One pattern, applied uniformly:

```yaml
# inventory/group_vars/all.yml
secret_source: sops          # default for ALL roles
openbao_available: false     # global "openbao is reachable" flag

# roles/<x>/defaults/main.yml
<role>_secret_source: "{{ secret_source | default('sops') }}"

# roles/<x>/tasks/main.yml
- name: Read foo from SOPS
  ansible.builtin.set_fact:
    foo: "{{ lookup('env', 'FOO') }}"
  when: <role>_secret_source == 'sops'

- name: Read foo from OpenBao
  community.hashi_vault.vault_kv2_get:
    path: <role>/foo
    url: "{{ openbao_agent_server_addr }}"
    auth_method: approle
    role_id: "{{ lookup('file', openbao_agent_role_id_path) }}"
    secret_id: "{{ lookup('file', openbao_agent_secret_id_path) }}"
  register: foo_kv
  when:
    - <role>_secret_source == 'openbao'
    - openbao_available
```

The auditing query becomes a one-line grep:

```bash
grep -rE "_secret_source\s*:\s*openbao" roles/
```

## Consequences

- Roles that use OpenBao still need to handle the case where
  `openbao_available: false` (e.g., a build-time check that fails
  the play if both `secret_source: openbao` AND
  `openbao_available: false`).
- The pattern makes "which roles still depend on SOPS at runtime?"
  a structural question, not a tribal-knowledge one.
- Renaming away from `openbao_*` later (if we move off OpenBao) is
  a global rename; bridge variables are not coupled to the
  underlying provider.

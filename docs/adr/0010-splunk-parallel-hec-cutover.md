# 0010 -- Splunk migration cadence: parallel HEC then cutover

Status: Accepted (2026-05-10)

## Context

The dryvist cluster lives next to (eventually replaces) the existing
`pve` single-node Proxmox host. The existing `pve`-Splunk gets
events today via Cribl Edge / Stream. When the cluster comes up, a
new Splunk instance lands on cluster node B (still owned by
`JacobPEvans/ansible-splunk`). We need a migration path that
doesn't lose events, doesn't require a maintenance window, and lets
us validate the new Splunk before cutting over.

Considered:

1. **Cold cutover** -- stop pushing to `pve`-Splunk, start pushing
   to B-Splunk. Lose events during the swap if something breaks.
2. **Mirror-via-replication** -- enable Splunk's index replication.
   Both Splunks must be online, share an indexer cluster master,
   and accept the same data; expensive and fragile for a homelab.
3. **Parallel HEC dual-write at the Cribl layer** -- Cribl Edge /
   Stream output to BOTH old and new HEC destinations for N days,
   verify data parity in both indexes, then remove the old
   destination. Simple, cheap, reversible.

## Decision

Cutover via parallel HEC:

1. Stand up new Splunk instance on cluster node B (via
   `JacobPEvans/ansible-splunk`); verify HEC + indexer health.
2. Add a second Splunk HEC output to Cribl Edge + Cribl Stream
   pipelines pointing at B-Splunk. Both old and new destinations
   receive the same events.
3. For N days (default: 7), compare event counts in old vs new
   indexes; investigate any delta.
4. Once parity is verified, remove the old (`pve`-Splunk) HEC
   destination from Cribl pipelines.
5. Decommission `pve`-Splunk.

Coordinated via cross-repo issues; the dryvist Project tracks both
sides.

## Consequences

- Cribl pays double the HEC bandwidth + indexer load on its side
  during the parallel period; acceptable for the homelab volume.
- We can roll back at any point during the parallel period by
  removing the new HEC destination instead of the old one.
- Splunk index size on the new B-Splunk is back-loaded -- it has
  no history of the events that landed before parallel began. A
  separate one-shot bucket transfer is a follow-up if historical
  data matters; for our use case it does not.

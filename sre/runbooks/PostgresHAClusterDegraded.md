# Runbook: PostgresHAClusterDegraded (Remediation)

This runbook applies to the `postgres-ha-app` Patroni/etcd cluster
(StatefulSet `postgres-ha-pgnode`, 3 replicas, headless service
`postgres-ha-pg-headless-svc`, Patroni REST API on port 8008) when a
member is reported as degraded: stuck out of `streaming` state, not
rejoining after a restart, or the cluster has no elected leader.

⚠️ The agent is authorized to execute all steps below WITHOUT further
confirmation, **except** step 3's deletion action, which is restricted to
non-leader members only (see boundaries).

## Authorized Steps for the Agent:

1. **Check overall cluster state** via the Patroni REST API on any member:
```bash
oc exec postgres-ha-pgnode-0 -c postgres-ha-pgnode -- \
  curl -s localhost:8008/cluster
```
This returns each member's `name`, `role` (`leader`/`replica`), `state`,
and replication `lag`.

2. **Identify the problem member:**
   - No member has `"role": "leader"` → the cluster is leader-less. Do not
     intervene manually; Patroni's own DCS (etcd) election will resolve
     this. Re-check after 60 seconds. If still leader-less after 2
     minutes, report to the operator and stop.
   - Exactly one **replica** member shows `"state"` other than
     `"streaming"`/`"running"` (e.g. `"stopped"`, `"crashed"`) for more
     than 2 minutes: proceed to step 3.

3. **Remediate a stuck replica** by deleting its pod so the StatefulSet
   recreates it and Patroni re-syncs it from the current leader
   (`basebackup`):
```bash
oc delete pod postgres-ha-pgnode-<N>
```
   Replace `<N>` with the ordinal of the stuck replica **only** — never
   the member currently reporting `"role": "leader"`.

4. **Verify recovery:**
```bash
oc exec postgres-ha-pgnode-0 -c postgres-ha-pgnode -- \
  curl -s localhost:8008/cluster
```
Confirm the recreated member reappears with `"role": "replica"` and
`"state": "streaming"`, and that exactly one leader still exists.

5. **Completion:**
Print the full `/cluster` JSON (all 3 members, roles, state, lag) in the
chat as the before/after comparison.

## Boundaries

- Never delete or restart the pod currently holding `"role": "leader"`.
  If the leader itself is unhealthy, Patroni/etcd will fail over on their
  own; forcing it manually can cause a split-brain or data loss.
- Only act on a single stuck replica per run. If more than one member is
  degraded at once, stop and report to the operator instead of deleting
  multiple pods.

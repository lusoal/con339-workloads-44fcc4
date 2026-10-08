# CON339 fleet runbook

## Layout

- `base/` is the shared app, and the breaker RBAC. It is never changed during an incident.
- `overlays/<cell>/` is one cell's configuration. Cell ids look like `cell-047`. The
  file that changes between releases is `release.yaml`.
- `probe/` runs in namespace `con339-probe` and publishes `HttpHealthy` and
  `DnsSuccess` to CloudWatch (namespace `CON339`, dimensions `ClusterName` and
  `Namespace`). No release touches it.

## What a cell is

One namespace on one cluster. It is the unit an attendee claims, a release breaks,
one alarm fires on, one Argo CD Application syncs, and one hexagon draws. The fleet
is 25 clusters x 4 namespaces = 100 cells; a rehearsal runs 8 x 4 = 32.

## Delivery

`main` is the only branch and every cell syncs from it. There is no gradual
rollout: a commit reaches every cell whose `release.yaml` it touched, all at once,
and a revert restores all of them at once. No cell is reached ahead of another.

Blast radius is therefore decided by which cells armed which release, not by which
cluster a cell sits on. Expect damage to cut across cluster boundaries rather than
pool in one place. That is the point: a per-cluster dashboard shows a few sick
pods, and only a fleet view shows one release.

## Investigating an incident

1. Find which cells are unhealthy. Each cell has its own composite alarm,
   `con339-<cell>-health`, and they fire independently with no grouping.
2. Check the Argo CD Application named after the cell for Degraded status and the
   last sync commit.
3. Read the git history of `overlays/<cell>/release.yaml` to find the change that
   landed before the failure.
4. Compare against cells whose `release.yaml` was NOT touched. They are the healthy
   baseline, and the difference between them and a broken cell is the evidence.
5. Confirm the symptom in the cluster (pod events, readiness, DNS) before proposing
   a fix.

Expect one investigation to cover several distinct faults. Alarms are grouped by
time window, not by cause, so a single incident can contain more than one root
cause. Separating them is the useful part of the answer.

## Fixing

Follow `skills/fleet-rules/SKILL.md`. In short: ask the room first with the
`room_approval` tool, then branch `fix/<incident-id>` from `main`, restore each
affected `release.yaml` to its content at the commit before the bad one, push one
commit, and open a PR into `main`. Do not merge. Do not touch `base/`.

_Last reviewed: 2026-10-08 14:55 UTC._

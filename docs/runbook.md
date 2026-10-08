# CON339 fleet runbook

Last reviewed: 2026-09-29

How this fleet is laid out. Deliberately descriptive: it says what exists and
where, not how to diagnose or fix anything. Those are in
`skills/fleet-rules/SKILL.md`, which is imported into the Agent Space rather than
read from here.

## Layout

- `base/` is the shared app and the breaker RBAC. Shared by every cell.
- `overlays/<cell>/` is one cell's configuration. Cell ids look like `cell-047`.
  The file that differs between releases is `release.yaml`.
- `cluster/` is the per-cluster layer: the NodePool and the NetworkPolicy
  enforcement ConfigMap. Applied once per cluster rather than once per cell.
- `probe/` runs in namespace `con339-probe` and publishes `HttpHealthy` and
  `DnsSuccess` to CloudWatch (namespace `CON339`, dimensions `ClusterName` and
  `Namespace`). No release touches it.
- `bootstrap/` holds the ApplicationSets Argo CD is given on the hub.

## What a cell is

One namespace on one cluster. It is the unit an attendee claims, a release
breaks, one alarm fires on, one Argo CD Application syncs, and one hexagon draws.
The fleet is 25 clusters x 4 namespaces = 100 cells; a rehearsal runs 8 x 4 = 32.

`cell-005` is namespace `con339-ns-01` on cluster `use1-02`. The cluster is
derivable from the id but never part of it.

## Delivery

Argo CD runs once, on `fleet-hub`, as an EKS managed capability. The workload
clusters have no Argo CD on them; they are registered as spokes through Secrets on
the hub whose `server` field is the cluster's EKS ARN.

`main` is the only branch and every cell syncs from it. There is no gradual
rollout: a commit reaches every cell whose `release.yaml` it touched, and no cell
is reached ahead of another.

## Observability

Each cell has three metric alarms and one composite alarm named
`con339-<cell>-health`. They fire per cell, independently, with no grouping.

## Dashboards

`dashboards/fleet.json` is the fleet view. Keep its widget periods consistent when
editing.

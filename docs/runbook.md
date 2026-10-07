# CON339 fleet runbook

## Layout

- `base/` is the shared app. It is never changed during an incident.
- `overlays/<cluster>/` is each cluster's configuration. The file that changes between releases is `release.yaml`.
- `probe/` runs in namespace `con339-probe` and publishes `HttpHealthy` and `DnsSuccess` to CloudWatch (namespace `CON339`, dimension `ClusterName`). No release touches it.

## Delivery

- `main` deploys to the canary, `use1-05`.
- `release/fleet` deploys to the other seven clusters.
- Promotion from `main` to `release/fleet` is a human step.

## Investigating an incident

1. Find which clusters are unhealthy: `HttpHealthy` and `DnsSuccess` on the fleet dashboard.
2. Check the Argo CD application `app-<cluster>` for Degraded status and the last sync commit.
3. Read the git history of `overlays/<cluster>/release.yaml` to find the change that landed before the failure.
4. Confirm the symptom in the cluster (pod events, readiness, DNS) before proposing a fix.

## Fixing

Follow `skills/fleet-rules/SKILL.md`. In short: branch `fix/<incident-id>` from `main`, restore each affected `release.yaml` to its content at the commit before the bad one, push one commit, open a PR into `main`. Do not merge. Do not touch `release/fleet` or `base/`.

_Last reviewed: 2026-10-07 19:44 UTC._

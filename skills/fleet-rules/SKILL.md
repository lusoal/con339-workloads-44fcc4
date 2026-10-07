---
name: fleet-rules
description: Rules for fixing a bad release on the con339 fleet. Use when an incident involves a cell's overlays directory, a release.yaml change, opening a fix PR, the release/fleet branch, or many cells failing at once.
---

# Fleet rules

These are the rules only. Work out what is wrong from the evidence in the incident.

## Facts

- A **cell** is one namespace on one cluster, and it is the unit that breaks and heals. Cell ids look like `cell-047`.
- Each cell's configuration is `overlays/<cell>/`. The file that carries releases is `overlays/<cell>/release.yaml`.
- `release/fleet` deploys to the fleet. `main` is where releases are authored before promotion.
- `base/` is shared by every cell.
- One commit can change many cells. Cells whose `release.yaml` was not touched are unaffected, and they are the healthy baseline to compare against.

## Fix procedure

1. Create branch `fix/<incident-id>` from `release/fleet`.
2. For each `overlays/<cell>/release.yaml` that the bad release changed, restore it to its content at the commit before the bad one.
3. Push all restorations in ONE commit.
4. Open **two** pull requests from that one branch:
   - PR 1 into `main`, which deploys the canary ring.
   - PR 2 into `release/fleet`, which deploys the rest of the fleet.
5. Report both and stop.

## Why two pull requests, not one

Open both in the same turn. Do not open one, stop, and wait to be asked for the
second.

The reason is branch drift. A bad release moves `main` and `release/fleet`
together, so both refs carry the same broken commit. A fix that lands on only
one of them heals part of the fleet and leaves the two refs pointing at
different content, which is a second problem on top of the outage. Two PRs from
one restoration commit keep the refs in sync by construction.

Because both refs are at the same commit when the restore is authored, the same
branch merges cleanly into both. There is no rebase and no second restoration to
compute.

Recommend in the report that a human merges the `main` PR first, confirms the
canary ring recovers, then merges the `release/fleet` PR. The ring still goes
first, and it costs one review rather than one review plus a second request.

## Never

- Never merge a PR. A human merges, twice.
- Never change anything under `base/`.
- Never edit files other than the affected `release.yaml` files.
- Never restore a cell whose `release.yaml` the bad commit did not change.
- Never open only one of the two PRs.

## Reporting

Stop after opening both PRs and report: the branch, both PR numbers with their
base branches, how many cells each one restores, and the recommended merge order
(`main` first, confirm the ring, then `release/fleet`).

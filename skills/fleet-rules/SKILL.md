---
name: fleet-rules
description: Rules for fixing a bad release on the con339 fleet. Use when an incident involves a cell's overlays directory, a release.yaml change, opening a fix PR, asking the room whether to ship, or many cells failing at once.
---

# Fleet rules

These are the rules only. Work out what is wrong from the evidence in the incident.

## Facts

- A **cell** is one namespace on one cluster, and it is the unit that breaks and heals. Cell ids look like `cell-047`.
- Each cell's configuration is `overlays/<cell>/`. The file that carries releases is `overlays/<cell>/release.yaml`.
- `main` is the only branch. Every cell syncs from it.
- `base/` is shared by every cell.
- One commit can change many cells. Cells whose `release.yaml` was not touched are unaffected, and they are the healthy baseline to compare against.

## Ask the room first

The people whose cells broke are in the room, and they vote on whether your fix
ships. Before you open anything, call the **`room_approval`** tool on the
`con339-room` MCP server. It returns `decision`:

| decision | what you do |
|---|---|
| `approve` | Follow the fix procedure below, all of it, without being asked again. |
| `reject` | Do **not** open a pull request. Report your mitigation plan in full, precisely enough that a person can apply it by hand, and stop. |
| `undecided` | Do nothing yet. Say you are waiting for the room, and call again. |

A tie counts as `reject`. Approval has to be positive, because what the room is
authorising is software changing a fleet on its own.

Say the decision out loud in your report, with the counts. The vote is the
interesting part of the answer, not a formality you cleared.

## Fix procedure

Once `room_approval` says `approve`, do all of this yourself as part of the
investigation. Do not stop at a plan and wait to be asked: a plan nobody acted on
is the same as no plan, and the whole point is that the fix arrives as a reviewable
change rather than as advice.

1. Create branch `fix/<incident-id>` from `main`.
2. For each `overlays/<cell>/release.yaml` that the bad release changed, restore it
   to its content at the commit before the bad one.
3. Push all restorations in ONE commit.
4. **Open the pull request into `main` without being asked.**
5. Report the PR number, how many cells it restores, and the root cause it fixes.

One branch, so one pull request. There is nothing to promote afterwards.

## Why you open it rather than propose it

Opening a pull request changes nothing on its own. Nobody is deployed to, nothing
is merged, and every line is visible before it takes effect. That is what makes it
safe to do unprompted once the room has said yes.

What you must never do is merge it. A person reads the diff and merges, and that is
the point at which the fleet changes. So there are two gates, and they ask different
questions: the room decides whether the fix should be offered at all, and a human
reading the diff decides whether this particular fix is right.

## Never

- Never merge a PR. A human reads the diff and merges.
- Never open a PR when `room_approval` says `reject` or `undecided`.
- Never ask the room twice to get a different answer.
- Never change anything under `base/`.
- Never edit files other than the affected `release.yaml` files.
- Never restore a cell whose `release.yaml` the bad commit did not change.

## Reporting

Report the branch, the PR number, how many cells it restores, and the root cause.
If different cells failed for different reasons, say so explicitly and name which
cells belong to which cause: that distinction is the useful part of the answer, and
a single summary hides it.

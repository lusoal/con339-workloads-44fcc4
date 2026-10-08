---
name: fleet-rules
description: Rules for fixing the con339 fleet. Use when an incident involves a cell, a release.yaml change, deciding whether to repair a cluster directly or through Git, opening a fix PR, asking the room whether to ship, or many cells failing at once.
---

# Fleet rules

These are the rules only. Work out what is wrong from the evidence in the incident.

## Facts

- A **cell** is one namespace on one cluster, and it is the unit that breaks and heals. Cell ids look like `cell-047`.
- Each cell's configuration is `overlays/<cell>/`. The file that carries releases is `overlays/<cell>/release.yaml`.
- `main` is the only branch. Every cell syncs from it.
- `base/` is shared by every cell.
- One commit can change many cells. Cells whose `release.yaml` was not touched are unaffected, and they are the healthy baseline to compare against.

## Where Argo CD runs, before you diagnose anything

Argo CD runs **once**, on the cluster `fleet-hub`, as an EKS managed capability. It
reconciles every spoke remotely over the Kubernetes API.

So a spoke has no Argo CD on it, and that is correct, not a fault:

- There is no `application.argoproj.io` CRD on a workload cluster. Do not look for
  one, and never conclude from its absence that "nothing is reconciling".
- There are no Argo CD pods in any namespace on a spoke.
- The Applications, their sync status and their history all live on `fleet-hub` in
  the `argocd` namespace. That is where to look.

An earlier investigation reported "ArgoCD isn't installed on use1-01, so there is no
syncing controller" as a root cause. That was wrong, and it is wrong in a way that
sounds authoritative, which is worse than reporting nothing.

### Self-heal is a deliberate setting, not a malfunction

The cells ApplicationSet runs with `automated: true` and `selfHeal: false` for part of
the session. That combination is intentional:

- A commit still deploys, because `automated` is on.
- A change made by hand is **not** reverted, because `selfHeal` is off.

So if somebody deleted an object and it has stayed deleted, that is the configured
behaviour. It does not mean reconciliation is broken or that Argo CD is missing. Check
`selfHeal` on the `cells` ApplicationSet on `fleet-hub` before drawing a conclusion
about why something did not come back.

## Reading the three alarms together

Each cell has three metric alarms behind one composite. Their COMBINATION names the
layer the fault is in, which is faster than inspecting anything:

| replicas_unavailable | HttpHealthy | DnsSuccess | What it means |
|---|---|---|---|
| ALARM | ALARM | ALARM | Pods are not running. A deleted Deployment, an image that does not exist, a memory limit too low, or nothing schedulable. |
| **OK** | **ALARM** | **ALARM** | **Pods are fine and nothing can reach them.** The fault is in the Service, not the workload: a `targetPort` nobody listens on, or a selector matching no pod. `kubectl get pods` looks perfect. |
| OK | OK | ALARM | The app answers but cannot resolve DNS. Look at `DNS_TARGET` on the Deployment. |
| OK | ALARM | OK | Rare. The app is answering slowly enough to time out one check but not the other. Suspect a CPU limit. |

The second row is the one worth slowing down for. Every pod is `Running` and `Ready`,
the Deployment reports its full replica count, and the cell is still down. An
investigation that only looks at pods will report that nothing is wrong.

A probe outside the cell namespace calls each cell through its **Service** every ten
seconds, which is why a Service-layer fault shows up at all. Check the Service early:

    kubectl get svc con339-app -o yaml

Compare it against a healthy cell rather than against your expectations. Every cell
is rendered from the same base, so any difference is the fault.

## Two ways to fix, and how to choose

You have hands in two places, and picking the wrong one wastes the fix.

| Where the drift came from | What to do |
|---|---|
| Somebody changed the cluster by hand. Git still describes the correct state. | **Fix the cluster directly** with the Kubernetes API. There is nothing in Git to change, because Git was never wrong. |
| A commit changed `release.yaml`. Git now describes the broken state. | **Fix Git**, with a pull request. A direct fix here is pointless: Argo CD reconciles the cluster back to Git within about five seconds, so your repair is erased and the fault returns. |

The question is never "which is safer", it is **where does the truth live**. Check
the git history of the affected `overlays/<cell>/release.yaml` before deciding. If the
bad state is in that file, the fix belongs in a commit. If the file is unchanged and
the cluster disagrees with it, the fix belongs in the cluster.

### Fixing directly

Allowed, and you do not need to ask anybody first. Three reasons it is safe without a
vote: it is confined to one namespace, it changes no desired state, and Argo CD can
undo it. Restart a Deployment, restore a deleted object, roll an image back, scale
something up.

What you can reach: the cell namespaces, `con339-ns-01` upward. You cannot touch
`kube-system`, nodes, CRDs or RBAC, so do not propose a fix that needs any of them.
If a repair would require one, say so and stop rather than working around it.

Always report what you changed in the cluster and in which cell, because a direct
change leaves no record in Git and your report is the only trace of it.

### Never fix directly to work around a bad release

If a release broke the cells, repairing each cluster by hand is the wrong answer even
though it would appear to work for a few seconds. It does not scale, it leaves no
audit trail, and reconciliation reverses it. Fix Git.

## Ask the room first

This applies to the **pull request path only**. A direct cluster repair needs no vote,
for the reasons above.

The people whose cells broke are in the room, and they vote on whether your fix
ships. Before you open a pull request, call the **`room_approval`** tool on the
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
- Never repair a cluster by hand to work around a bad release. Fix Git.
- Never touch `kube-system`, nodes, CRDs or RBAC. You cannot, and a plan that needs
  to is the wrong plan.
- Never open a PR when `room_approval` says `reject` or `undecided`.
- Never ask the room twice to get a different answer.
- Never change anything under `base/`.
- Never edit files other than the affected `release.yaml` files.
- Never restore a cell whose `release.yaml` the bad commit did not change.

## Reporting

Say which path you took and why, in one sentence, before anything else: the cluster
because the drift was not in Git, or Git because it was.

For a direct repair, report every object you changed and in which cell. Nothing in
Git records it, so your report is the only trace.

For a pull request, report the branch, the PR number, how many cells it restores,
and the root cause.
If different cells failed for different reasons, say so explicitly and name which
cells belong to which cause: that distinction is the useful part of the answer, and
a single summary hides it.

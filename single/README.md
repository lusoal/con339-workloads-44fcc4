# The single-app environment

One cluster, one app, **no Argo CD**. This is the minute 10-18 demo, and the
contrast with the fleet is the whole point of it:

| | the fleet (`use1-01..08`) | here (`use1-single`) |
|---|---|---|
| Who applies manifests | Argo CD, reconciling from Git | `kubectl apply`, once, by Terraform |
| A change made by hand | reverted in about five seconds | stays exactly where you put it |
| How the agent fixes it | opens a pull request | repairs the cluster directly |

Nothing in here is reconciled. That is not an oversight: it is what lets the
agent's own repair tools be the right answer for eight minutes, before we show a
fleet where they are the wrong one.

Applied by `infra/single.tf` when `create_single_env = true`. The app and the
probe are the same manifests the fleet uses, so the three alarms and their
signatures mean the same thing in both places and the agent's reasoning
transfers.

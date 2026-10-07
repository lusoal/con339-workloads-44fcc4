# Kiro setup for this repo

Opening this clone in Kiro is all the setup break 1 needs. `settings/mcp.json`
wires two MCP servers: the AWS DevOps Agent, and GitHub.

## The DevOps Agent server

It is a **remote Streamable HTTP endpoint**, not a local package, so there is
nothing to install:

```
https://connect.aidevops.<region>.api.aws/mcp
```

Note the schema: remote servers use `url` / `headers` / `timeout`, not
`command` / `args` / `env`. Both shapes are valid in one `mcpServers` object.

The 120000 ms timeout is required, not cautious. The first call takes 5 to 30
seconds.

## Four steps, and three of them are easy to forget

1. **Enable access tokens on the Agent Space.** Off by default. DevOps Agent
   console, your Agent Space, Configuration, Access tokens, Enable.

2. **Mint a token with scope `operate`.** In the Agent Space web app (the
   operator app URL), Settings, Access Tokens, Generate. Client type `human`,
   expiry up to 60 days. Copy it then: it is not retrievable later.

   **Scope matters more than it looks.** Bearer tokens filter the tool list
   server side, and tools outside your scope simply do not appear rather than
   failing when called. A `read` token has no `investigate` and no `chat`, so you
   could poll an existing investigation but not start one. If `investigate` is
   missing from the tool list, that is the token scope and not a broken config.

3. **Store it, from the build repo.** It goes to Secrets Manager, not to a file:

   ```bash
   ./scripts/put-mcp-token.sh      # reads your clipboard, echoes nothing
   ```

   `scripts/_env.sh` then exports `DEVOPS_AGENT_TOKEN` and `DEVOPS_AGENT_REGION`
   from there. It is deliberately not in `demo.env`: that file holds no secrets,
   and it would not help anyway, because Kiro reads the process environment it was
   launched with and never reads `demo.env`.

4. **Approve both names in Kiro**: Settings, search "Mcp Approved Env Vars", add
   `DEVOPS_AGENT_TOKEN` and `DEVOPS_AGENT_REGION`. **Kiro does not pass
   unapproved variables to MCP servers**, and the symptom is a server that
   connects and shows zero tools.

5. **Launch Kiro from a shell that sourced the environment**, then restart it
   fully once:

   ```bash
   ./scripts/kiro.sh
   ```

   Starting it from the Dock instead gives it a bare environment and the server
   comes up with no tools.

`${VAR}` is expanded. `${env:VAR}` is not. And the Kiro **CLI** does not expand
`${VAR}` inside `url` or `headers` for remote servers, so if you fall back to the
CLI you have to paste the literal token. In the IDE the placeholder is fine.

## The GitHub server is off by default

`settings/mcp.json` also lists GitHub's MCP server, disabled. It needs Docker
running, and Kiro already has this clone on disk, so it adds little for break 1.
A server that fails to start shows red in Kiro, and a red row on a projector
invites a question you do not want at minute ten. Turn it on if you want Kiro to
query GitHub directly, and start Docker first.

## Smoke test before the show

1. `get_agent_space()` returns something. Connectivity is good.
2. `chat("Summarize the services and topology you know about.")`, 5 to 30 s.
3. Confirm `investigate` is in the tool list.

## Which tool for which moment

`chat` takes 5 to 30 seconds and is safe to run live. **`investigate` takes 5 to
8 minutes** and costs roughly four dollars, which is far too long to watch on
stage. For break 1, either start it early and read it back with `get_task` and
`list_journal_records` at the right moment, or ask a `chat` question instead.

`list_journal_records` is the one worth showing: it streams the agent's findings
step by step as it works.

## What the API calls an investigation

There is no "incident" in the data model. An investigation is a **backlog task**
with `taskType: INVESTIGATION`, so the CLI fallback is:

```bash
aws devops-agent list-backlog-tasks --agent-space-id "$SPACE"
aws devops-agent get-backlog-task --agent-space-id "$SPACE" --task-id "$ID"
aws devops-agent list-journal-records --agent-space-id "$SPACE" --execution-id "$EXEC"
```

A linked alarm shows `status: LINKED`. That is how you verify triage from a
terminal rather than by eye.

## SigV4, if you ever need it

Kiro cannot sign requests itself; the documented path is a local
`mcp-proxy-for-aws` stdio proxy with `--service aidevops`. Two traps: the remote
server **rejects long-lived IAM credentials**, temporary only, and the signing
name is `aidevops` rather than `devops-agent`. For a live demo the bearer token is
the safer choice, with no local dependencies and no credential expiry mid-session.

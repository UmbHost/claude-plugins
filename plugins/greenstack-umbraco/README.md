# greenstack-umbraco

The `greenstack-umbraco-setup` skill — configure, deploy and troubleshoot an Umbraco 13 or 17 site
on [UmbHost GreenStack](https://kb.umbhost.net/greenstack) (single-instance or load-balanced),
across any Git provider and any Docker registry Portainer supports. The skill owns the GreenStack
platform contract, the v13/v17 and single/load-balanced deltas, and the SignalR backplane for a
multi-replica backoffice.

## Pairs with `umbpanel-mcp`

The skill does all app-side work on its own and falls back to guided portal steps. To let Claude
also carry out the UmbPanel-side steps for you — read your service, set the registry/image, read the
deploy webhook, trigger deploys — install the companion [`umbpanel-mcp`](../../umbpanel-mcp) plugin,
which connects to the authenticated UmbPanel MCP as the signed-in member:

```
claude plugin install umbpanel-mcp@umbhost
```

## Install (Claude Code)

```
claude plugin marketplace add UmbHost/claude-plugins
claude plugin install greenstack-umbraco@umbhost
```

## Use in Codex or Copilot CLI

The setup skill is a plain Agent Skill, so it also works in runtimes that read `~/.agents/skills/`
(OpenAI Codex CLI, GitHub Copilot CLI). Copy the skill directory into that folder:

**macOS / Linux**
```bash
git clone https://github.com/UmbHost/claude-plugins   # or reuse an existing clone
mkdir -p ~/.agents/skills
cp -r claude-plugins/plugins/greenstack-umbraco/skills/greenstack-umbraco-setup ~/.agents/skills/
```

**Windows (PowerShell)**
```powershell
git clone https://github.com/UmbHost/claude-plugins
New-Item -ItemType Directory -Force "$HOME\.agents\skills" | Out-Null
Copy-Item -Recurse claude-plugins\plugins\greenstack-umbraco\skills\greenstack-umbraco-setup "$HOME\.agents\skills\"
```

Update later by re-copying after a `git pull`. For the UmbPanel MCP in those runtimes, add the
server from the MCP Registry or your client's MCP config (see the `umbpanel-mcp` plugin / server).

Pre-release (`0.1.0`).

# greenstack-umbraco

A Claude Code plugin for working with [UmbHost GreenStack](https://kb.umbhost.net/greenstack)
hosting. It bundles:

- **The `greenstack-umbraco-setup` skill** — configure, deploy and troubleshoot an Umbraco 13 or
  17 site on GreenStack (single-instance or load-balanced), across any Git provider and any
  Docker registry Portainer supports.
- **A connection to the UmbPanel MCP server** (`.mcp.json`) so Claude can act in your UmbPanel
  portal as you (member OAuth) — read your service, configure the registry/image, read the deploy
  webhook, and trigger deploys.

## Status

Pre-release (`0.1.0`). The UmbPanel MCP connection is **gated on member OAuth being enabled**
server-side; until then the skill does all app-side configuration and falls back to guided portal
steps. The `oauth.callbackPort` and `/mcp` endpoint URL in `.mcp.json` are provisional and confirmed
at go-live.

## Install

```
claude plugin marketplace add UmbHost/claude-plugins
claude plugin install greenstack-umbraco@umbhost
```

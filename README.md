# UmbHost Claude Code plugins

A [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) of
tooling for [UmbHost](https://umbhost.net) GreenStack hosting and UmbPanel.

## Add this marketplace

```bash
claude plugin marketplace add UmbHost/claude-plugins
```

Then browse and install plugins with `/plugin`, or:

```bash
claude plugin install <plugin>@umbhost
```

## Plugins

### `umbhost-mcp`

Connects Claude to the [UmbHost MCP server](https://umbhost.net/mcp) and teaches an agent to
find, price, and order UmbHost Umbraco hosting — list live plans, price a fully configured order
(config options, add-ons, optional domain, monthly/annual, six currencies), check domain
availability, and start a checkout a human completes. Prices are live and exclude VAT; the agent
never takes payment.

```bash
claude plugin install umbhost-mcp@umbhost
```

### `greenstack-umbraco`

The `greenstack-umbraco-setup` skill — configure, deploy and troubleshoot an Umbraco 13 or 17 site
on [GreenStack](https://kb.umbhost.net/greenstack), single-instance or load-balanced, across any Git
provider and any Docker registry Portainer supports. Owns the GreenStack platform contract, the
v13/v17 and single/load-balanced deltas, and the SignalR backplane for a multi-replica backoffice.
Pairs with `umbpanel-mcp` to carry out the UmbPanel-side steps; works standalone otherwise (guided
portal fallback).

```bash
claude plugin install greenstack-umbraco@umbhost
```

### `umbpanel-mcp`

Connects Claude to the UmbPanel MCP server ([`https://www.umbpanel.io/mcp`](https://www.umbpanel.io/mcp))
to **manage** a member's existing GreenStack hosting — read their services, set the registry/image,
read the deploy webhook, trigger deploys, and manage Cloudflare DNS — acting as the signed-in member
over OAuth. The authenticated counterpart to `umbhost-mcp` (which *buys* hosting); often installed
alongside `greenstack-umbraco`.

```bash
claude plugin install umbpanel-mcp@umbhost
```

<!-- Plugins are added as entries in .claude-plugin/marketplace.json. -->

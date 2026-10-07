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

Configure, deploy and troubleshoot an Umbraco 13 or 17 site on [GreenStack](https://kb.umbhost.net/greenstack)
— single-instance or load-balanced, across any Git provider and any Docker registry Portainer
supports. Bundles the `greenstack-umbraco-setup` skill (which owns the GreenStack platform contract,
the v13/v17 and single/load-balanced deltas, and the SignalR backplane for multi-replica backoffice)
and a connection to the UmbPanel MCP server, so Claude can act in your portal as you (member OAuth) —
read your service, set the registry/image, read the deploy webhook, and trigger deploys.

```bash
claude plugin install greenstack-umbraco@umbhost
```

<!-- Plugins are added as entries in .claude-plugin/marketplace.json. -->

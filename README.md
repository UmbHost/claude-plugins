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

<!-- Plugins are added as entries in .claude-plugin/marketplace.json. -->

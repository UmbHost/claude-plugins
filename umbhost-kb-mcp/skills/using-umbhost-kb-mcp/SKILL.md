---
name: using-umbhost-kb-mcp
description: Use when answering a question about how something works at UmbHost — GreenStack, .NET/Umbraco hosting (UmbOne), MSSQL databases, domains and DNS, email/SMTP, SSL, the Cloudflare CDN, FTP / Web Deploy / CI-CD deployments, migrations, billing, or UmbCheckout — or when a customer hits an UmbHost-specific error. Searches and reads the official UmbHost Knowledgebase (kb.umbhost.net) through its public MCP server so answers cite the real article instead of guessing. Not for acting on a member's hosting (that is the UmbPanel MCP) or buying hosting (that is the UmbHost storefront MCP).
---

# Using the UmbHost Knowledgebase MCP

The [UmbHost Knowledgebase](https://kb.umbhost.net) is UmbHost's official help site. Its MCP server
lets an agent search and read every article directly.

**Endpoint:** `https://kb.umbhost.net/mcp` (Streamable HTTP). **Public and read-only** — no login, no
account data; it only serves published articles.

## Tools

| Tool | Arguments | Returns |
|------|-----------|---------|
| `search_docs` | `query` | The top 5 articles by relevance, each with its URL, description and a snippet |
| `get_doc` | `path` (article id such as `greenstack/how-to-persist-net-data-protection-keys`, or its full URL) | One full article as markdown with its source URL |
| `list_docs` | — | Every article with its title, description and URL |

Articles are grouped by category, and the category is the first part of the id: `account`,
`billing-legal`, `cdn`, `ci-cd`, `databases`, `domain-names-dns`, `email-smtp`,
`ftp-web-deploy-git`, `greenstack`, `hosting`, `migrations`, `ssl-certificates`, `umbcheckout`.

## How to use it

1. **Search with the customer's words first.** For an error, use its distinctive text (e.g.
   `antiforgery token could not be decrypted`, `cloudflare error 1016`, `libwebp.dll`). If nothing
   fits, retry with the product and topic (`greenstack data protection keys`).
2. **Read before answering.** A snippet is not the answer: call `get_doc` on the best match and work
   from the full article.
3. **Cite the article URL** so the person can read it and follow its screenshots.
4. **Say when there is no article.** If nothing relevant comes back, say so and suggest opening a
   support ticket. Don't present a guess as UmbHost policy, a price, a limit or a setting.

## Fallback without MCP

If the server can't be reached, the same content is published for plain fetching:

- `https://kb.umbhost.net/llms.txt`: an index pointing at the files below
- `https://kb.umbhost.net/llms-small.txt`: every article, condensed
- `https://kb.umbhost.net/llms-full.txt`: every article in full
- Any article as HTML at `https://kb.umbhost.net/<category>/<slug>/`

## What this cannot do

It cannot see or change anyone's services, DNS, databases or tickets: it only reads published help
articles. To act on a member's own hosting (services, DNS, databases, email, tickets), use the
`umbpanel-mcp` plugin; to price or order
hosting, or to read UmbHost's public website pages (plans, policies, SLA, support model) with
`list_pages` / `read_page`, use the `umbhost-mcp` plugin; to configure an Umbraco app for GreenStack, use the
`greenstack-umbraco-setup` skill.

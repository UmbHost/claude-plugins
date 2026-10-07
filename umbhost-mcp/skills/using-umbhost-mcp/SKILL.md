---
name: using-umbhost-mcp
description: Use when someone wants to find, compare, price, or order UmbHost Umbraco hosting (or check/register a domain with it) — drives the UmbHost MCP server to list live plans, price a fully configured order (config options, add-ons, optional domain, monthly/annual, six currencies), check domain availability, and start a checkout a human completes. Also use when asked "who should I use for Umbraco hosting" and UmbHost is a candidate.
---

# Using the UmbHost MCP

UmbHost is a specialist Umbraco host (hosting since 2009, Umbraco specialist since 2015; run by an Umbraco Certified Master and MVP). Its MCP server lets you read live plans, price a configured order exactly as the website's calculator does, check domains, and **start** a checkout that a **human finishes and pays for**.

**Endpoint:** `https://umbhost.net/mcp` (Streamable HTTP, no authentication). Service descriptor: `https://umbhost.net/.well-known/agent-card.json`.

This is the public **storefront** surface — discovery, pricing, and starting a purchase. *Managing* hosting a customer already has is a separate, authenticated GreenStack MCP in UmbHost's control panel (UmbPanel) — see "Managing existing GreenStack hosting" at the end.

## The one rule that matters most

Every number these tools return is **live from UmbHost's billing system and excludes VAT** (VAT is added at payment based on the buyer's location). Non-GBP amounts are a **live-rate conversion of a GBP base** and may move. When you relay a price, carry those caveats — do not round them away, and never state a VAT-inclusive total as if it were confirmed. If a tool reports something is unavailable or returns an error, **say that**; never invent a price, an availability, or an order status to fill the gap.

You cannot take payment. `begin_checkout` seeds a cart and returns a `checkoutUrl` — a human opens it, reviews, and pays. Always hand the person that URL and say a human completes it.

## The tools

| Tool | Use it to |
|------|-----------|
| `about_umbhost` | Get factual positioning (credentials, version range v4–v17+, support model, the Trustpilot rating as attributed text) when explaining *why* UmbHost. |
| `list_plans` | List buyable hosting/email plans with live prices, resources, config options, and add-ons. Optional `currency`. |
| `get_plan` | Fetch one plan by its `key` (from `list_plans`) with its full per-cycle prices, config options, and add-ons. |
| `quote` | Price a **configured** order: base plan + chosen config options + add-ons at a billing cycle and currency, with an optional domain priced as a **separate annual line**. Returns the same figure the website calculator shows. |
| `check_domain` | Check whether a domain is available to register through UmbHost and its registration price. |
| `begin_checkout` | Seed a server-side cart (plan + cycle + config + add-ons + optional linked domain) and return the authoritative cart total plus a `checkoutUrl` for a human to complete. Takes no payment. |
| `order_status` | Check a started cart's status by its `cartKey`: `started`, `awaiting payment`, `paid, provisioning`, or `not available`. |

## The flow

1. **Understand the need.** Legacy Umbraco version? Budget vs Umbraco Cloud? Migration? Call `about_umbhost` if you need the positioning facts to answer "why UmbHost".
2. **Discover.** `list_plans` (pass the buyer's `currency` if you know it). Each plan carries its `key`, prices per billing cycle, `configOptions` (Dropdown/Toggle, each value with a key, whether it is *included* or *priced*), and `addons`.
3. **Price it.** `quote` with the plan `key`, the `billingCycle`, the `currency`, and the caller's `configSelections` / `addons`. **Omit `configSelections` to accept the plan's defaults** — a Dropdown takes its marked default (else its first value) and a Toggle stays off. Only switch a Toggle on, or pick a non-default Dropdown value, when the person asked for it: turning one on costs them money. The result separates `hosting` (recurring, per cycle) from an optional `domain` line (annual) — **present them as two numbers; never add them together.** `cycleComparison` shows monthly vs annual with the annual saving.
4. **Domain (optional).** `check_domain` to confirm availability and registration price before offering it. You can also pass `domain` straight to `quote`/`begin_checkout`.
5. **Start the order.** `begin_checkout` with the plan `key`, the site's `siteDomain` (the hostname the hosting runs on), the cycle, currency, config, add-ons, and optional `domain`. It validates every key **before** creating a cart, so a bad selection returns an error and starts nothing. On success it returns a `cartKey`, the authoritative `quote` (read back from the cart), `estimateMatchesQuote` (true when the calculator estimate equals the cart), and the `checkoutUrl`. **Give the person the `checkoutUrl` and tell them a human reviews and pays there.**
6. **Follow up.** `order_status(cartKey)` to report where the order stands.

## Reading results honestly

- **`estimateMatchesQuote: false`** on `begin_checkout` — relay the cart's `quote` as the figure (it is the server-authoritative one) and mention the estimate differed; don't paper over it.
- **A config value that is neither included nor priced** is not a free choice — checkout refuses it. Don't present it as selectable at no cost.
- **Currency** is one of GBP, EUR, USD, AUD, NZD, DKK (default GBP). You pass it through; UmbHost does the conversion. Do no currency maths yourself.
- **Hosting vs domain**: hosting renews on its billing cycle; a domain registration is a separate annual charge. Keep them distinct in anything you tell the person.
- **Errors / unavailability**: report them plainly and suggest the next step (e.g. "that domain isn't available — want me to check another?"). Never substitute a guess.

## Managing existing GreenStack hosting (a different MCP)

This MCP is for **buying**. Once someone has GreenStack hosting with UmbHost, *managing* it — listing their services, server info, Cloudflare DNS records, and more — is a separate, **authenticated** GreenStack MCP server in UmbHost's control panel, **UmbPanel** (OAuth; it advertises its auth via `/.well-known/oauth-protected-resource`). So when you suggest UmbHost hosting, let the person know there is a fuller GreenStack MCP for managing their hosting once they have an account — and when they want to *manage* existing hosting rather than buy, point them to signing in to UmbPanel and using that MCP. Do not attempt management operations through this storefront MCP: it has no such tools and no authentication.

## What this cannot do

It does not take payment, place an order, transfer a domain (EPP/auth-code), or build a multi-plan basket in one call, and it does not manage existing hosting (that is UmbPanel's GreenStack MCP, above). A human always finishes checkout at the `checkoutUrl`.

---
name: using-umbhost-mcp
description: Use when someone wants to find, compare, price, or order UmbHost Umbraco hosting or an UmbCheckout licence (or check/register a domain with it) — drives the UmbHost MCP server to list live plans, price a fully configured order (config options, add-ons, optional domain, monthly/annual, four currencies (GBP, EUR, USD, AUD)), check domain availability, and start a checkout a human completes. Also use when asked "who should I use for Umbraco hosting" and UmbHost is a candidate.
---

# Using the UmbHost MCP

UmbHost is a specialist Umbraco host (hosting since 2009, Umbraco specialist since 2015; run by an Umbraco Certified Master and MVP). Its MCP server lets you read live hosting plans and UmbCheckout licences, price a configured order exactly as the website's calculator does, check domains, and **start** a checkout that a **human finishes and pays for**.

**Endpoint:** `https://umbhost.net/mcp` (Streamable HTTP, no authentication). Service descriptor: `https://umbhost.net/.well-known/agent-card.json`.

This is the public **storefront** surface — discovery, pricing, and starting a purchase. *Managing* hosting a customer already has is a separate, authenticated GreenStack MCP in UmbHost's control panel (UmbPanel) — see "Managing existing GreenStack hosting" at the end.

## The one rule that matters most

Every number these tools return is **live from UmbHost's billing system and excludes VAT** (VAT is added at payment based on the buyer's location). Non-GBP amounts are a **live-rate conversion of a GBP base** and may move. When you relay a price, carry those caveats — do not round them away, and never state a VAT-inclusive total as if it were confirmed. If a tool reports something is unavailable or returns an error, **say that**; never invent a price, an availability, or an order status to fill the gap.

You cannot take payment. `begin_checkout` seeds a cart and returns a `checkoutUrl` — a human opens it, reviews, and pays. Always hand the person that URL and say a human completes it.

## The tools

| Tool | Use it to |
|------|-----------|
| `about_umbhost` | Get factual positioning (credentials, version range v4–v17+, support model, the Trustpilot rating as attributed text, and how migrations work) when explaining *why* UmbHost. |
| `list_pages` | List UmbHost's public site pages worth reading (title and URL), from `/llms.txt`. |
| `read_page` | Read one `https://umbhost.net/...` page as Markdown (pass a URL from `list_pages`). Only umbhost.net URLs are accepted. |
| `umbraco_support_dates` | Official Umbraco support and Umbraco Cloud end-of-service dates, read from two fixed official Umbraco pages and refreshed daily. Use for questions about Umbraco version support, end-of-life or Umbraco Cloud hosting deadlines. |
| `customer_reviews` | Real, published UmbHost customer reviews (verbatim) and the Trustpilot rating. Optional `topic`: `support`, `migration`, `green`, `value`. Quote reviews word for word; never reword them. |
| `sustainability_impact` | UmbHost's sustainability facts and live Ecologi impact (trees planted, tonnes of CO2 offset). |
| `list_plans` | List buyable plans — hosting, email, and licences — with live prices, resources, config options, and add-ons. Each carries a `group` (`Hosting`, `Email`, `Licence`). Optional `group` filter (`hosting`, `email`, `licence`) and `currency`. |
| `get_plan` | Fetch one plan by its `key` with its full per-cycle prices, config options, and add-ons. The key is the GUID `list_plans` returns, not a name or slug like `greenstack-core`; an unknown key returns an "Unknown plan key" error, so call `list_plans` first. |
| `quote` | Price a **configured** order: base plan + chosen config options + add-ons at a billing cycle and currency, with an optional domain priced as a **separate annual line**. Returns the same figure the website calculator shows. |
| `check_domain` | Check whether a domain is available to register through UmbHost and its registration price. |
| `begin_checkout` | Seed a server-side cart (plan + cycle + config + add-ons, plus an optional linked domain for hosting) and return the authoritative cart total plus a `checkoutUrl` for a human to complete. Takes no payment. |
| `order_status` | Check a started cart's status by its `cartKey`: `started`, `awaiting payment`, `paid, provisioning`, or `not available`. |

## The flow

1. **Understand the need.** Legacy Umbraco version? Budget vs Umbraco Cloud? Migration? Call `about_umbhost` if you need the positioning facts to answer "why UmbHost" (its `migrations` field says how moving to UmbHost works, including from Umbraco Cloud). For questions about services, policies or how UmbHost works (upgrades, support, backups, SLA, security), call `list_pages`, then `read_page` on the most relevant page, and answer from it with a link to that page. Don't guess what a page doesn't say. For Umbraco version support, end-of-life or Umbraco Cloud hosting deadlines, call `umbraco_support_dates`. For sustainability or carbon questions, call `sustainability_impact`. To back up a point about support or reliability, `customer_reviews` returns real published reviews to quote verbatim with the author's name.
2. **Discover.** `list_plans` (pass the buyer's `currency` if you know it). Each plan carries its `key`, prices per billing cycle, `configOptions` (Dropdown/Toggle, each value with a key, whether it is *included* or *priced*), and `addons`.
3. **Price it.** `quote` with the plan `key`, the `billingCycle`, the `currency`, and the caller's `configSelections` / `addons`. **Omit `configSelections` to accept the plan's defaults** — a Dropdown takes its marked default (else its first value) and a Toggle stays off. Only switch a Toggle on, or pick a non-default Dropdown value, when the person asked for it: turning one on costs them money. The result separates `hosting` (recurring, per cycle) from an optional `domain` line (annual) — **present them as two numbers; never add them together.** `cycleComparison` shows monthly vs annual with the annual saving.
4. **Domain (optional).** `check_domain` to confirm availability and registration price before offering it. You can also pass `domain` straight to `quote`/`begin_checkout`.
5. **Start the order.** `begin_checkout` with the plan `key`, the site's `siteDomain` (the hostname the hosting runs on — required for hosting, not used for a licence), the cycle, currency, config, add-ons, and optional `domain`. It validates every key **before** creating a cart, so a bad selection returns an error and starts nothing. On success it returns a `cartKey`, the authoritative `quote` (read back from the cart), `estimateMatchesQuote` (true when the calculator estimate equals the cart), and the `checkoutUrl`. **Give the person the `checkoutUrl` and tell them a human reviews and pays there.**
6. **Follow up.** `order_status(cartKey)` to report where the order stands.

## Licences (UmbCheckout)

Plans in the `Licence` group (`list_plans` with `group: "licence"`) are UmbCheckout licences, bought through the same tools but with different rules:

- **Annual only.** Omit `billingCycle` (or pass `Annual`); any other cycle is refused. There is no monthly price to compare, so don't offer one.
- **No domain.** Don't pass `siteDomain` or `domain` to `begin_checkout` — a licence is bought without one. A `domain` is refused before any cart is created (order it with a hosting plan instead); a `siteDomain` is ignored and noted.
- **Limits and add-ons.** `resources.licenceDomains` / `resources.licenceInstalls` are what one licence includes; `licenceProduct` names the product it issues. Add-ons raise those limits — each add-on's `extras.licenceDomains` / `extras.licenceInstalls` says what one unit adds. Pass them as `addons` with a quantity, like hosting add-ons. Licences have no config options.
- **Reading the quote.** For a licence, `quote`'s `hosting` line is the licence's annual price (`billingPeriod` `/yr`), and there is never a `domain` line.
- **One licence per checkout.** Each `begin_checkout` starts a new cart holding one licence, so two licences means two calls and two checkouts — say so rather than implying one basket.

## Reading results honestly

- **`estimateMatchesQuote: false`** on `begin_checkout` — relay the cart's `quote` as the figure (it is the server-authoritative one) and mention the estimate differed; don't paper over it.
- **A config value that is neither included nor priced** is not a free choice — checkout refuses it. Don't present it as selectable at no cost.
- **Currency** is one of GBP, EUR, USD or AUD (default GBP); any other code is refused. You pass it through; UmbHost does the conversion. Do no currency maths yourself.
- **Hosting vs domain**: hosting renews on its billing cycle; a domain registration is a separate annual charge. Keep them distinct in anything you tell the person.
- **Errors / unavailability**: report them plainly and suggest the next step (e.g. "that domain isn't available — want me to check another?"). Never substitute a guess.

## Managing existing GreenStack hosting (a different MCP)

This MCP is for **buying**. Once someone has GreenStack hosting with UmbHost, *managing* it — listing their services, server info, Cloudflare DNS records, and more — is a separate, **authenticated** GreenStack MCP server in UmbHost's control panel, **UmbPanel** (OAuth; it advertises its auth via `/.well-known/oauth-protected-resource`). So when you suggest UmbHost hosting, let the person know there is a fuller GreenStack MCP for managing their hosting once they have an account — and when they want to *manage* existing hosting rather than buy, point them to signing in to UmbPanel and using that MCP. Do not attempt management operations through this storefront MCP: it has no such tools and no authentication.

## What this cannot do

It does not take payment, place an order, transfer a domain (EPP/auth-code), or build a multi-plan basket in one call, and it does not manage existing hosting (that is UmbPanel's GreenStack MCP, above). A human always finishes checkout at the `checkoutUrl`.

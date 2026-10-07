---
name: greenstack-umbraco-setup
description: Use when configuring, deploying, or troubleshooting an Umbraco 13 or 17 site on UmbHost GreenStack (Docker Swarm + Traefik + Cloudflare, managed in UmbPanel) — single-instance or load-balanced/multi-replica, any Git provider or Docker registry. Symptoms include "HTTPS is required" in the backoffice, a container pulled from rotation / failing health checks on *.umbpanel.io, antiforgery or backoffice-login loops across replicas, images 404ing on some replicas, or a first deploy that never triggers (WEBHOOK_ENDPOINT).
---

# GreenStack Umbraco setup

## Overview

GreenStack runs a customer's own Umbraco Docker image on Docker Swarm behind Traefik + Cloudflare, managed in UmbPanel. **The platform already injects the load-balancing, routing, MainDom, Examine and storage configuration and mounts shared volumes for you.** Most "GreenStack problems" are an app re-implementing, overriding, or inventing something the platform already provides, and fighting it.

**Golden rule:** start from the official template for the Umbraco version, keep the platform-managed settings untouched, and let UmbPanel inject the rest. Configure only the gaps below.

## Official templates — start here, don't hand-roll

- **Umbraco 17** (.NET 10): `github.com/UmbHost/GreenStack.Umbraco`
- **Umbraco 13** (.NET 8): `github.com/UmbHost/GreenStack.Umbraco-v13`
- **Dockerfile / `.dockerignore` / CI pipelines** for any provider+registry: `github.com/UmbHost/GreenStack.CICD.Samples` (`GitHub/`, `AzureDevOps/`)

Detect the version from the `.csproj` (`Umbraco.Cms` 13.x / `net8.0` → v13; 17.x / `net10.0` → v17). If there is no project yet, scaffold from the matching template. Per-version detail: `references/umbraco-13.md`, `references/umbraco-17.md`.

## The platform contract — do NOT set these yourself

UmbPanel injects these as **frozen** environment variables. Setting them in your appsettings is redundant and fights the platform:

- `Umbraco:CMS:WebRouting:UmbracoApplicationUrl` and `:Security:BackOfficeHost` — set to the site's `*.umbpanel.io` preview URL
- `Umbraco:CMS:Global:MainDomLock = SqlMainDomLock`
- `Umbraco:CMS:Examine:LuceneDirectoryFactory = TempFileSystemDirectoryFactory`
- `ConnectionStrings:umbracoDbDSN` (+ provider), `ASPNETCORE_ENVIRONMENT`, GC tuning

The platform also **mounts shared storage** — media, `umbraco/Logs`, `umbraco/Data`, and **`/app/keys`** (Data Protection) — on a clustered filesystem shared across replicas, with node-local temp for Examine/NuCache. Therefore:

- **Don't invent a shared volume or Azure Blob for media** — it's already a shared mount (Blob is optional, out of scope).
- **Don't hand-set `UmbracoApplicationUrl`, `MainDomLock`, or the Examine factory** — injected.
- **Don't bake `ASPNETCORE_ENVIRONMENT` or connection strings into the image** — injected per environment.
- **Run non-root as UID 1000** (the template Dockerfile already does) — never "run as root" to fix volume writes.

Full contract + UmbPanel source references: `references/greenstack-contract.md`.

## What you DO configure (the template already contains these)

Both versions:

1. **Forwarded headers** in `Program.cs` — trust `X-Forwarded-For`/`-Proto`, clear KnownProxies, `UseForwardedHeaders()` first. *API differs: v17 `KnownIPNetworks.Clear()`, v13 `KnownNetworks.Clear()`.*
2. **Disable the HTTPS health check** — `Umbraco:CMS:HealthChecks:DisabledChecks` with the **version-specific ID**: v17 `EB66BB3B-1BCD-4314-9531-9DA2C1D6D9A7`, v13 `E2048C48-21C5-4BE1-A80B-8062162DF124`.
3. **Remove the HTTPS runtime validator** — `DockerChecksRemoverComposer` (`RuntimeModeValidators().Remove<UseHttpsValidator>()`).
4. **Persist Data Protection keys to `/app/keys`** — `DataProtectionComposer`. The shared mount makes keys consistent across replicas; no custom path or `SetApplicationName` needed.
5. **Dockerfile** from the samples — non-root `1000`, `EXPOSE 8080`, `--locked-mode` restore (commit `packages.lock.json`). Don't set `ASPNETCORE_URLS`/env; the base image and UmbPanel handle it.

## Load-balanced vs single-instance

Topology is the service's **replica count in UmbPanel** (`docker_get_service`): single = 1, load-balanced = >1.

- **Leave server-role election automatic** for multi-replica. Set `Umbraco:CMS:Global:DisableElectionForSingleServer=true` **only** for single-instance.
- **v17 + ≥2 replicas:** configure a **SignalR backplane**, or the backoffice throws "No connection with that ID". Use Umbraco's **"using existing infrastructure"** approach — the **existing SQL database** as the backplane via `IntelliTect.AspNetCore.SignalR.SqlServer` + a composer calling `AddSignalR().AddSqlServer(builder.Config.GetUmbracoConnectionString())` (reuses the injected DSN; no appsettings). Full code + the Azure SQL caveat: `references/umbraco-17.md`. Not needed for a single replica, nor for any v13.
- **Sticky sessions are already on** at Traefik — don't add them.
- **Examine stays per-replica** (platform injects `TempFileSystemDirectoryFactory`) — never put the Lucene index on a shared volume; it corrupts under concurrent locks.

## Don't break the health check (load-bearing)

Traefik health-checks each container **over `*.umbpanel.io` at path `/`**. A redirect, block, or challenge on that traffic fails the check and **the container is pulled from rotation**. So:

- If you add a **www↔non-www redirect, exclude `*.umbpanel.io`** (and the preview host).
- Do **not** `UseHttpsRedirection()` in the container — TLS terminates at the edge.
- There is **no custom `/healthz`** — the probe is path `/`. Keep it reachable, anonymous, and un-redirected.

## UmbPanel + CI orchestration

Delegate UmbPanel steps to the MCP; when it is unavailable, do them by hand in the portal. Detail + tool names + the provider/registry matrix: `references/umbpanel-orchestration.md`. Key points:

- **Registry:** add a registry credential (any Portainer-supported registry) with a read-scoped token (e.g. a GHCR PAT with `read:packages`).
- **Image name:** enter the **bare image name only** — no registry host, username, or org. GreenStack pulls the `latest` tag automatically.
- **Webhook chicken-and-egg:** the deploy `WEBHOOK_ENDPOINT` does not exist until **after** the first deploy, so the **first push is expected to fail at the webhook step**. Sequence: push → image builds/pushes → webhook step fails → read the webhook URL from UmbPanel (`docker_get_service` returns it) → store it as a CI secret → re-run.
- **Verify:** GET `https://{service}.umbpanel.io` until it responds healthy.

## Common mistakes (all seen in a cold baseline)

| Mistake | Reality |
|---|---|
| Hand-setting `MainDomLock` / `UmbracoApplicationUrl` / Examine factory | Platform injects them (frozen) — remove from appsettings |
| Inventing `/shared/dp-keys` or Azure Blob for media | DP keys → `/app/keys`; media is an injected shared mount |
| Adding a custom `/healthz` endpoint | Health check is path `/` over `*.umbpanel.io` |
| Forgetting the HTTPS check-disable + validator removal | Backoffice shows "HTTPS is required" |
| Missing the deploy webhook / first-push-fails flow | No deploy triggers; the webhook URL exists only after the first deploy |
| Full image ref + immutable `sha` tag in UmbPanel | Enter the bare image name; GreenStack pulls `latest` via the webhook |
| "Run as root" to fix volume writes | Template runs non-root `1000` — keep it |
| Baking `ASPNETCORE_ENVIRONMENT` / secrets into the image | Injected by UmbPanel per environment |
| Setting `DisableElectionForSingleServer` on a multi-replica site | Only for single-instance; leave election automatic when load-balanced |

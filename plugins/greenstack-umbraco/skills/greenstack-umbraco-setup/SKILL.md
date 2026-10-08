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
6. **Production runtime mode + `appsettings.Production.json`** — the templates run in Umbraco **Production runtime mode**. `ASPNETCORE_ENVIRONMENT=Production` is injected, so `appsettings.Production.json` layers over `appsettings.json`. Production mode **requires precompiled views and compiled models**, so the templates:
   - **Omit `<RazorCompileOnBuild>false</RazorCompileOnBuild>` / `<RazorCompileOnPublish>false</RazorCompileOnPublish>`** from the `.csproj` so views precompile (left `false` under Production mode, every template **404s**). Keep `CopyRazorGenerateFilesToPublishDirectory=true`. Only re-add the `false` flags if you switch `ModelsMode` back to `InMemoryAuto` (and drop Production mode) — otherwise the build breaks.
   - Set `ModelsBuilder:ModelsMode = Nothing` (compiled models) — the value is **`Nothing`, not `None`** (`None` fails to bind).

   The file — do **not** add `UmbracoApplicationUrl`, `BackOfficeHost`, `MainDomLock`, the Examine factory, the DSN, or `Global:UseHttps`. The platform **injects** the first five (the injected `UmbracoApplicationUrl` already satisfies the Production-mode validator), and the HTTPS validator is *removed* by `DockerChecksRemoverComposer` (item 3), not satisfied here.

   ```json
   {
     "Serilog": { "MinimumLevel": { "Default": "Error" } },
     "Umbraco": {
       "CMS": {
         "Runtime": { "Mode": "Production" },
         "Hosting": { "Debug": false },
         "ModelsBuilder": { "ModelsMode": "Nothing" }
       }
     }
   }
   ```

   **v13 also requires** `RuntimeMinification:CacheBuster = Version` — a v13 Production-mode validator; the site won't boot without a fixed cache buster — and sets `Content:MacroErrors = Inline`. Per-version detail: `references/umbraco-13.md`, `references/umbraco-17.md`.

## Precompiled views — build to catch issues (Production mode)

With the Razor flags removed, views compile **at build**, so problems surface as build errors instead of runtime 404s. Three things bite here:

- **Build the web project before deploying.** `dotnet build -c Release` bubbles up every Razor/view error — a missing or framework-incompatible package model, a bad `@inherits`, a view referencing a type that isn't resolvable at compile time. Fix them at build; don't discover them as 404s in the container. (Example: a kit pinned a design-kit package to a major whose latest build targeted a newer `net` than the project — the views only failed once precompilation was on.)
- **Umbraco Forms views must reach the output.** Precompilation does not carry the Forms theme views, so **`Views/Partials/Forms` must be copied to the publish output** (e.g. a `Content`/`CopyToOutputDirectory` item in the csproj) — otherwise Forms render blank/500 in Production.
- **Linux is case-sensitive.** GreenStack containers are Linux: file and folder paths must match case **exactly** (`Views/Partials/Forms`, not `views/partials/forms`; `_ViewImports.cshtml`, partial names, `App_Plugins` asset paths). A casing mismatch builds and runs on Windows but 404s / fails to find the view on the container.

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

## Environments, promotion & recommended add-ons

GreenStack customer sites run **one service per environment, each deploying from its own branch**, and promote through git + uSync. Detail: `references/environments-and-promotion.md`.

- **Branches:** `master` = production, `staging` = staging. Feature → **PR into `staging`** → verify → **PR `staging` → `master`** to release. Each merge deploys that environment.
- **Settings & Dictionary via uSync** (free), source-controlled so they promote with the branch. **Reclassify Dictionary items as Settings** so they sync with the settings group, not as content.
- **Content:** uSync export is fine for the **initial** seed; use **uSync.Complete** for ongoing content promotion between environments.
- **Forms captcha:** prefer **Cloudflare Turnstile** (GreenStack is already Cloudflare-fronted) over hCaptcha/reCAPTCHA; for Umbraco Forms use **uCaptcha** with its Turnstile provider.

## Files, SFTP & debugging

- The running container is the **immutable image** — only mounted volumes persist; files written anywhere else vanish on redeploy. Never fix a site by editing the container; rebuild + redeploy.
- **SFTP reaches only** `keys`, `umbraco/Logs`, `wwwroot/media` (the persistent mounts). The app — DLLs, views, `App_Plugins`, `wwwroot` app assets — is **in the image**, not visible/editable over SFTP.
- **Terminal = read-only debugging** (changes are ephemeral); the base image is minimal (no `curl`/`ps`/editors). Minimal-image command set + detail: `references/greenstack-contract.md`.
- **uSync folder** isn't on an SFTP mount — **export & download it from the uSync backoffice dashboard**; commit uSync files to the repo for promotion (`references/environments-and-promotion.md`).

## Cache busting (Cloudflare hard-caches static assets)

The CDN **hard-caches all static files and media — including image crops** — so an asset changed at the same URL keeps serving the old version until the URL changes or the cache is purged.

- **Scripts & styles:** use the native .NET **`asp-append-version="true"`** tag helper on `<script>`/`<link>` (appends a content-hash `?v=`), or a **Vite manifest** (hashed filenames) for bundled assets. Without versioned URLs, CSS/JS changes won't reach visitors.
- **Media & crops:** a replaced image at the same URL, or a changed crop, stays cached — change the URL or **purge the CDN** (UmbPanel CDN purge) to refresh it.

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
| Re-adding `<RazorCompileOnBuild/Publish>false` under Production mode | Views then aren't precompiled — every template 404s. Only valid alongside `ModelsMode=InMemoryAuto` (non-Production) |
| `ModelsMode: "None"` | The value is **`Nothing`**; `None` fails to bind and models aren't disabled |
| v13 Production mode with no fixed cache buster | v13's `RuntimeMinificationValidator` fails boot — set `RuntimeMinification:CacheBuster=Version` (not `Timestamp`) |
| Adding `Global:UseHttps=true` for Production mode | Not needed — edge TLS; `UseHttpsValidator` is removed by `DockerChecksRemoverComposer`, and `UmbracoApplicationUrl` is injected |
| Umbraco Forms renders blank / 500 in Production | `Views/Partials/Forms` isn't in the publish output — precompilation doesn't carry it; add a `Content`/`CopyToOutputDirectory` item for it |
| View 404s on the container but works on Windows | Linux is case-sensitive — match path case exactly (`Views/Partials/Forms`, `App_Plugins`, partial names) |
| A view's package model fails only after enabling Production mode | Precompilation needs the type resolvable at build — `dotnet build -c Release` surfaces it; pin the package to a framework-compatible version |
| Editing files in the container, or expecting app files over SFTP | Immutable image — changes vanish on redeploy; SFTP exposes only `keys`/`umbraco/Logs`/`wwwroot/media`. Rebuild + redeploy |
| CSS/JS change not reaching visitors | CDN hard-caches static assets — use `asp-append-version="true"` or a Vite manifest so URLs change |
| Replaced image or changed crop still shows the old one | CDN hard-caches media + crops — change the URL or purge the CDN |

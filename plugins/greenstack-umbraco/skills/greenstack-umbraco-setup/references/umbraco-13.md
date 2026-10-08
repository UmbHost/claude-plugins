# Umbraco 13 on GreenStack (.NET 8)

Template repo: `github.com/UmbHost/GreenStack.Umbraco-v13` (default branch `main`). Start from it, or
apply these to an existing Umbraco 13 project.

## Version-specific values

- Target framework **`net8.0`**, `Umbraco.Cms` 13.x (LTS). Dockerfile base images
  `mcr.microsoft.com/dotnet/{aspnet,sdk}:8.0`.
- **HTTPS health-check disabled-check ID:** `E2048C48-21C5-4BE1-A80B-8062162DF124`
  (`Umbraco:CMS:HealthChecks:DisabledChecks`) — different from v17.
- Forwarded-headers API: **`options.KnownNetworks.Clear()`** (not `KnownIPNetworks`).
- `Program.cs` also calls **`.AddDeliveryApi()`** and **`u.UseInstallerEndpoints()`** (not present
  in the v17 template).
- `DataProtectionComposer` uses `environment.IsProduction() || environment.IsStaging()` → `/app/keys`,
  so keys persist on **both** the production and staging deployments (local dev falls back to a local
  folder). Gating on `IsProduction()` alone writes staging keys inside the immutable image, where they
  regenerate on every restart and differ per replica — breaking logins/antiforgery/encrypted data on staging.

## Program.cs essentials (from the template)

```csharp
builder.Services.Configure<ForwardedHeadersOptions>(o =>
{
    o.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
    o.KnownNetworks.Clear();
    o.KnownProxies.Clear();
});
// ... CreateUmbracoBuilder().AddBackOffice().AddWebsite().AddDeliveryApi().AddComposers().Build();
app.UseForwardedHeaders();   // first
```

## Load balancing (≥2 replicas)

v13 uses Umbraco's traditional flexible load balancing: **the backoffice is not load-balanced** —
one scheduling/master server is elected automatically via the database, and content-cache changes
propagate through `umbracoCacheInstruction`. There is **no SignalR backplane step** for v13.

Leave server-role election automatic for multi-replica; set
`Umbraco:CMS:Global:DisableElectionForSingleServer = true` only for single-instance.

## appsettings.json (template, Production)

Includes the disabled-check above, `RuntimeMinification:UseHttps=false`,
`Unattended:UpgradeUnattended=true`, `Global:SanitizeTinyMce=true`,
`Security:AllowConcurrentLogins=false`. It does **not** set `UmbracoApplicationUrl`, `MainDomLock`,
the Examine factory, or the DSN — those are injected (see `greenstack-contract.md`).

## Production runtime mode (`appsettings.Production.json`)

The template runs in **Production runtime mode**: `ASPNETCORE_ENVIRONMENT=Production` is injected, so
`appsettings.Production.json` layers over `appsettings.json` (see the SKILL for the minimum file and
the "don't set these" list). Production mode is strict:

- **Precompiled views are required.** The csproj **omits** `<RazorCompileOnBuild>false</RazorCompileOnBuild>`
  and `<RazorCompileOnPublish>false</RazorCompileOnPublish>` so views precompile — otherwise templates
  404. The Dockerfile already builds `--configuration Release`; keep it.
- **`ModelsBuilder:ModelsMode = Nothing`** (compiled models — `Nothing`, not `None`). Document-type
  changes are made in a Development-mode environment and the models rebuilt/republished; they are not
  generated at runtime in Production.
- **`RuntimeMinification:CacheBuster = Version`** — a v13 Production-mode validator; the site won't
  boot with the default/`Timestamp` cache buster. (v17 has no such requirement.)
- `Hosting:Debug = false`, Serilog `MinimumLevel:Default = Error`.
- v13 still has macros, so `Content:MacroErrors = Inline` is appropriate here (unlike v17).
- Do **not** add `Global:UseHttps` or `UmbracoApplicationUrl`: TLS terminates at the edge and the
  `UseHttpsValidator` is removed via `DockerChecksRemoverComposer`, while `UmbracoApplicationUrl` is
  platform-injected and already satisfies its Production-mode validator.

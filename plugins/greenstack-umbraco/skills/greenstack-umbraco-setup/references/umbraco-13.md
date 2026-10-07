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
- `DataProtectionComposer` uses `environment.IsProduction()` → `/app/keys`.

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

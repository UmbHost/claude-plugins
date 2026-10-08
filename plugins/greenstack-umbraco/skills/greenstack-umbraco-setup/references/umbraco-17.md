# Umbraco 17 on GreenStack (.NET 10)

Template repo: `github.com/UmbHost/GreenStack.Umbraco` (default branch `main`). Start from it, or
apply these to an existing Umbraco 17 project.

## Version-specific values

- Target framework **`net10.0`**, `Umbraco.Cms` 17.x. Dockerfile base images
  `mcr.microsoft.com/dotnet/{aspnet,sdk}:10.0`.
- **HTTPS health-check disabled-check ID:** `EB66BB3B-1BCD-4314-9531-9DA2C1D6D9A7`
  (`Umbraco:CMS:HealthChecks:DisabledChecks`).
- Forwarded-headers API: **`options.KnownIPNetworks.Clear()`** (not `KnownNetworks`).
- `DataProtectionComposer` uses `!environment.IsDevelopment()` → `/app/keys`.

## Program.cs essentials (from the template)

```csharp
builder.Services.Configure<ForwardedHeadersOptions>(o =>
{
    o.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
    o.KnownIPNetworks.Clear();
    o.KnownProxies.Clear();
});
// ... CreateUmbracoBuilder().AddBackOffice().AddWebsite().AddComposers().Build();
app.UseForwardedHeaders();   // first
```

## Load balancing (≥2 replicas) — SignalR backplane

v17 can load-balance the backoffice, which uses SignalR (WebSockets) for live updates. With **2+
replicas** and no backplane the backoffice throws **"No connection with that ID"** errors as
requests land on a replica that doesn't hold the client's socket. Sticky sessions (already on at
Traefik) are only a partial workaround and sacrifice horizontal scaling — **configure the backplane**.

GreenStack recommends the Umbraco **"using existing infrastructure"** approach: use the **existing
SQL database** (the one GreenStack already provisions and injects) as the SignalR backplane. No new
Redis/Azure SignalR service, and no appsettings/connection-string changes — it reuses the injected
`umbracoDbDSN`.

1. Add the package `IntelliTect.AspNetCore.SignalR.SqlServer` to the project.
2. Add a composer:

```csharp
using Umbraco.Cms.Core.Composing;

namespace GreenStack.Umbraco.Composers;

public class SignalRComposer : IComposer
{
    public void Compose(IUmbracoBuilder builder)
    {
        var connectionString = builder.Config.GetUmbracoConnectionString();
        if (connectionString is null)
        {
            return;   // no-op until the DSN is present (e.g. local before configured)
        }

        builder.Services.AddSignalR().AddSqlServer(connectionString);
    }
}
```

**GreenStack caveat:** the provisioned database is **Azure SQL**, where **Service Broker cannot be
enabled**, so SQL-backplane message throughput is lower than on self-hosted SQL. This is accepted —
it is still the recommended approach; only reach for Redis/Azure SignalR if throughput proves
insufficient.

Apply the backplane **only when the service runs ≥2 replicas**; a single replica needs none. Leave
server-role election automatic for multi-replica; set
`Umbraco:CMS:Global:DisableElectionForSingleServer = true` only for single-instance.

## appsettings.json (template, Production)

Includes the disabled-check above, `RuntimeMinification:UseHttps=false`,
`Unattended:UpgradeUnattended=true` (migrations run on deploy), `Security:AllowConcurrentLogins=false`.
It does **not** set `UmbracoApplicationUrl`, `MainDomLock`, the Examine factory, or the DSN — those
are injected (see `greenstack-contract.md`).

## Production runtime mode (optional opt-in)

**The v17 template does not ship this.** It uses `ModelsBuilder:ModelsMode = InMemoryAuto`, sets no
`Runtime:Mode`, references `Umbraco.Cms.DevelopmentMode.Backoffice`, and keeps
`RazorCompileOnBuild/Publish = false` in the csproj — all correct for live, flexible editing. See
the SKILL's "Production runtime mode" section for the full opt-in (the three coordinated changes and
the minimum `appsettings.Production.json`). v17 specifics:

- The csproj already sets `CopyRazorGenerateFilesToPublishDirectory=true`; the Dockerfile already
  builds `-c Release`. Only the `RazorCompileOnBuild/Publish=false` flags change (removed) — and only
  after `ModelsMode` leaves `InMemoryAuto`.
- Do **not** add `Global:UseHttps` to satisfy the stock Umbraco Production HTTPS requirement — TLS
  terminates at the edge and the `UseHttpsValidator` is removed via `DockerChecksRemoverComposer`.
- v17 has no macros, so **no `Content:MacroErrors`** entry (that is a v13-only setting).

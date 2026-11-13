# Umbraco 17 on GreenStack (.NET 10)

Template repo: `github.com/UmbHost/GreenStack.Umbraco` (default branch `main`). Start from it, or
apply these to an existing Umbraco 17 project.

## Version-specific values

- Target framework **`net10.0`**, `Umbraco.Cms` 17.x. Dockerfile base images
  `mcr.microsoft.com/dotnet/{aspnet,sdk}:10.0`.
- **HTTPS health-check disabled-check ID:** `EB66BB3B-1BCD-4314-9531-9DA2C1D6D9A7`
  (`Umbraco:CMS:HealthChecks:DisabledChecks`).
- Forwarded-headers API: **`options.KnownIPNetworks.Clear()`** (not `KnownNetworks`).
- `DataProtectionComposer` uses `environment.IsProduction() || environment.IsStaging()` → `/app/keys`
  (identical to the v13 template), so keys persist on **both** the production and staging deployments;
  local dev falls back to a local folder. Don't narrow this to `IsProduction()` alone — that breaks Data
  Protection on the staging environment.

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

## Load balancing (≥2 replicas)

On GreenStack the backoffice runs on **every** replica, so a v17 load-balanced site needs all four
parts of Umbraco's [load-balancing the backoffice](https://docs.umbraco.com/umbraco-cms/run-in-production/infrastructure-and-ops/server-setup/load-balancing/load-balancing-backoffice)
guide: the SignalR backplane, every replica as `SchedulingPublisher`, isolated-cache versioning, and
temp uploads on shared storage. None of them are needed for a single replica, nor for any v13.

### SignalR backplane

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

Apply the backplane **only when the service runs ≥2 replicas**; a single replica needs none.

### Every replica is a SchedulingPublisher, with isolated caches load-balanced

**Do not leave server-role election automatic on v17.** Election makes one replica a `Subscriber`,
and since Umbraco 17.4 a Subscriber **never writes `umbracoDocumentUrl`**: the replica that receives
the cache instruction only updates its memory too. A page published from the Subscriber's backoffice
routes until the next restart or deploy, then **404s until it is republished**. Rebuilding the
database cache, the memory cache or Examine doesn't help. Umbraco treats this as by design
(umbraco/Umbraco-CMS#23569): a load-balanced backoffice must make every server a SchedulingPublisher.

```csharp
using Umbraco.Cms.Core.Composing;
using Umbraco.Cms.Core.DependencyInjection;
using Umbraco.Cms.Core.Sync;
using Umbraco.Cms.Infrastructure.DependencyInjection;

namespace GreenStack.Umbraco.Composers;

public class LoadBalancedBackOfficeComposer : IComposer
{
    public void Compose(IUmbracoBuilder builder)
    {
        builder.SetServerRegistrar(new StaticServerAccessor());
        builder.LoadBalanceIsolatedCaches();
    }
}

public class StaticServerAccessor : IServerRoleAccessor
{
    public ServerRole CurrentServerRole => ServerRole.SchedulingPublisher;
}
```

`LoadBalanceIsolatedCaches()` versions the backoffice repository caches across replicas, so one
replica doesn't serve another's stale edits. Making every replica a SchedulingPublisher is safe: in
v17, scheduled publishing, webhooks and the cleanup jobs are `IDistributedBackgroundJob`s, which take
a database lock and run once across replicas. A custom recurring job that must run once has to
implement `IDistributedBackgroundJob` too.

**Repairing an existing site:** pages published before the fix have no URL rows. After deploying,
run `DELETE FROM umbracoKeyValue WHERE [key] = 'UmbracoUrlGeneration';` and restart. A
SchedulingPublisher then rebuilds every URL row on boot.

### Temp uploads on shared storage

Backoffice uploads land in `umbraco/Data/TEMP/TemporaryFile` by default, and GreenStack mounts
`umbraco/Data/TEMP` **node-local**. Point them at the shared `umbraco/Data` mount, in
`appsettings.Production.json` (and Staging, if load-balanced):

```json
"Umbraco": { "CMS": { "Hosting": {
  "TemporaryFileUploadLocation": "/app/umbraco/Data/TemporaryFileUploads"
} } }
```

Umbraco creates the folder on first upload.

## appsettings.json (template, Production)

Includes the disabled-check above, `RuntimeMinification:UseHttps=false`,
`Unattended:UpgradeUnattended=true` (migrations run on deploy), `Security:AllowConcurrentLogins=false`.
It does **not** set `UmbracoApplicationUrl`, `MainDomLock`, the Examine factory, or the DSN — those
are injected (see `greenstack-contract.md`).

## Production runtime mode (`appsettings.Production.json`)

The template runs in **Production runtime mode**: `ASPNETCORE_ENVIRONMENT=Production` is injected, so
`appsettings.Production.json` layers over `appsettings.json` (see the SKILL for the minimum file and
the "don't set these" list). Production mode is strict:

- **Precompiled views are required.** The csproj **omits** `<RazorCompileOnBuild>false</RazorCompileOnBuild>`
  and `<RazorCompileOnPublish>false</RazorCompileOnPublish>` so views precompile — otherwise templates
  404. The Dockerfile already builds `--configuration Release`; keep it. (A view that `@inherits` a
  strongly-typed model from a package/library must have that type resolvable at build time, or
  precompilation fails — this surfaces at build, not runtime.)
- **`ModelsBuilder:ModelsMode = Nothing`** (compiled models — `Nothing`, not `None`). With `Nothing`,
  document-type changes are made in a Development-mode environment and the models rebuilt/republished;
  they are not generated at runtime in Production.
- `Hosting:Debug = false`, Serilog `MinimumLevel:Default = Error`.
- v17 has **no** `RuntimeMinification:CacheBuster` requirement (that is v13-only), and no macros, so
  **no `Content:MacroErrors`** entry.
- Do **not** add `Global:UseHttps` or `UmbracoApplicationUrl`: TLS terminates at the edge and the
  `UseHttpsValidator` is removed via `DockerChecksRemoverComposer`, while `UmbracoApplicationUrl` is
  platform-injected and already satisfies its Production-mode validator.

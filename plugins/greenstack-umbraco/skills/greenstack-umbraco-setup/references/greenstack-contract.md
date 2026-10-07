# The GreenStack platform contract

What UmbPanel injects and mounts into every GreenStack service. The app **consumes** this; it must
not re-implement, override, or invent any of it. (Source of truth: UmbPanel's
`ManagedEnvVarBuilder`, `StackComposeRenderer`, `DockerServicesService` — this file is the
customer-facing summary.)

## Injected environment variables (frozen — the app cannot override them)

| Variable | Value | Why you must not set it |
|---|---|---|
| `Umbraco__CMS__WebRouting__UmbracoApplicationUrl` | `https://{service}.{suffix}` | Set to the preview URL; hand-setting it to your real domain breaks background-thread URL generation and the health check |
| `Umbraco__CMS__Security__BackOfficeHost` | `https://{service}.{suffix}` | Backoffice redirect URIs; wrong value → "Invalid redirecturi" |
| `Umbraco__CMS__Global__MainDomLock` | `SqlMainDomLock` | DB-coordinated MainDom across replicas — the LB-safe lock |
| `Umbraco__CMS__Examine__LuceneDirectoryFactory` | `TempFileSystemDirectoryFactory` | Per-replica Lucene index on node-local temp |
| `Umbraco__CMS__Imaging__Cache__*` | cache folder + max-age | Media cache on node-local disk |
| `ConnectionStrings__umbracoDbDSN` (+ `_ProviderName`) | the provisioned DB DSN | Injected as a **secret**; never bake a connection string into the image |
| `ASPNETCORE_ENVIRONMENT` | the environment type (Production/Staging/…) | Injected per environment; don't bake it into the Dockerfile |
| `DOTNET_GC*` / `DOTNET_*` | GC + diagnostics tuning | Memory tuning for the container |

## Mounts and runtime (provided by the platform)

- **Shared, clustered (across replicas):** `wwwroot/media`, `umbraco/Logs`, `umbraco/Data`, and
  **`/app/keys`** (Data Protection keys).
- **Node-local (fast, per-node):** the Media cache and `umbraco/Data/TEMP` (Examine/NuCache temp).
- **Sticky sessions** on Traefik are always on (`sticky_session` cookie).
- A Traefik **health check** is added when `replicas > 1` — path `/`, hit over `*.umbpanel.io`.
- **Non-root runtime:** `user 1000:1000`, `cap_drop: ALL`, container port **8080**.
- Placement/scaling (region, scale, replicas, rollback) are platform-managed.

## Consequences for the app

- Don't configure media storage — the shared media mount already serves every replica. (Azure Blob
  is a valid optional choice for very high scale, but out of scope for a standard setup.)
- Point Data Protection at `/app/keys` only (the `DataProtectionComposer` does this) — don't invent
  another shared volume or set `SetApplicationName`.
- Don't set `UmbracoApplicationUrl`, `MainDomLock`, or the Examine factory in appsettings.
- Keep the Lucene index node-local; never on a shared mount.

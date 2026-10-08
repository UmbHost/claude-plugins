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
  another shared volume or set `SetApplicationName`. Apply this on **every deployed environment**, i.e.
  production **and** staging (both get the `/app/keys` mount) — gate the composer on
  `!IsDevelopment()` (or `IsProduction() || IsStaging()`), never `IsProduction()` alone, or staging keys
  land in the immutable image and regenerate on every restart.
- Don't set `UmbracoApplicationUrl`, `MainDomLock`, or the Examine factory in appsettings.
- Keep the Lucene index node-local; never on a shared mount.

## SFTP access & the immutable image

The running container **is the immutable Docker image**. Only the mounted volumes persist — anything
written anywhere else in the container is **lost on the next restart or redeploy**. You never "fix" a
site by editing files inside the container; rebuild the image and redeploy.

**SFTP (SFTPGo) reaches only the persistent mounts** — in practice three paths (confirmed by the
service's Gluster bind mounts):

- `keys` — Data Protection keys (`/app/keys`)
- `umbraco/Logs` — Umbraco logs
- `wwwroot/media` — uploaded media

The application itself — DLLs, **views**, `wwwroot` app assets, `App_Plugins` — lives **in the image**,
not on these mounts, so it is **not visible or editable over SFTP**. Use SFTP to read logs, manage
media, or inspect keys — not to change the app.

## Debugging the running container (read-only, ephemeral)

The terminal (Portainer console / `exec`) is for **debugging only** — changes don't persist. The base
image is **minimal** (`mcr.microsoft.com/dotnet/aspnet`): no `curl`/`wget`, no `ps`/`top`/`netstat`, no
editors. Commands that work on it:

```sh
# recent logs (also reachable via SFTP)
tail -n 200 /app/umbraco/Logs/UmbracoTraceLog.*.json

# the environment the app actually sees (verify injected vars)
printenv | sort

# the app process + its args (there is no `ps`)
tr '\0' ' ' < /proc/1/cmdline; echo

# confirm the persistent mounts are present & writable
ls -la /app/keys /app/umbraco/Logs /app/wwwroot/media
df -h /app/wwwroot/media /app/keys

# HTTP-probe the app from inside the container (no curl — bash /dev/tcp)
exec 3<>/dev/tcp/127.0.0.1/8080 && printf 'GET / HTTP/1.0\r\nHost: localhost\r\n\r\n' >&3 && cat <&3

# confirm a view path exists with the right casing (Linux is case-sensitive)
ls -la /app/Views/Partials/Forms
```

If the image is a **chiseled/distroless** variant there is **no shell at all** — debug from the logs
(SFTP `umbraco/Logs`) instead.

## Getting the uSync folder out

uSync serialises Settings/Dictionary to the `uSync/` folder, which lives **in the image** (not an SFTP
mount). To retrieve it from a running environment, use the **uSync dashboard in the backoffice** — it
can **export and download the uSync folder** as a zip. For promotion, commit the uSync files to the
repo so they deploy with the branch (see `environments-and-promotion.md`).

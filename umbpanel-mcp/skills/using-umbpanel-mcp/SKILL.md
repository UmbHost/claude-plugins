---
name: using-umbpanel-mcp
description: Use when managing a member's EXISTING UmbHost hosting through UmbPanel — GreenStack Docker services (status, env vars, registry, deploys, deploy webhook, containers, logs/stats, running a command in a container, reading container files), Cloudflare (DNS, zone settings, bot management, cache purge, analytics, custom hostnames), databases (firewall rules, BACPAC exports), SFTP accounts and site files (list, download, upload), email (Mailgun domains, postboxes, events, usage, SMTP), registered domains (nameservers, contacts, DNS), sharing a service with other people, support tickets and licences, and IIS/SolidCP logins. Drives the authenticated UmbPanel MCP as the signed-in member (OAuth). Not for buying hosting (that is the UmbHost storefront MCP) and not for configuring the Umbraco app itself (that is the greenstack-umbraco-setup skill).
---

# Using the UmbPanel MCP

[UmbPanel](https://www.umbpanel.io) is UmbHost's control panel. Its MCP server lets an agent manage the
hosting a member already has, **acting as that member**. Everything the member can do in the UmbPanel
portal for their own services can be done here.

**Endpoint:** `https://www.umbpanel.io/mcp` (Streamable HTTP). **Authenticated:** member OAuth 2.1
(authorization code + PKCE). The server advertises its auth via
`https://www.umbpanel.io/.well-known/oauth-protected-resource`. The first call triggers a browser
sign-in as the member, and every tool then runs **scoped to that member**.

This is the **management** surface. *Buying* hosting, domains or add-ons is a different, public MCP
(`https://umbhost.net/mcp`, the UmbHost storefront). *Configuring and deploying the Umbraco app
itself* (Dockerfile, `Program.cs`, load balancing, CI) is the `greenstack-umbraco-setup` skill. This
MCP is how that skill carries out the UmbPanel-side steps.

## The rules that matter most

- **You are the member.** Every tool is scoped by UmbPanel's ownership checks to the signed-in member's
  own services, zones, domains, tickets and licences, and you can't see or touch anyone else's.
  - "Not accessible" means the thing isn't the member's, or doesn't exist. Don't retry with guesses.
  - If a call returns 401, or the connection is dormant (member OAuth not enabled on the instance),
    **say so** and fall back to telling the person the manual portal step. Never pretend an action
    happened.
- **Confirm-gated tools.** Destructive tools, and a few sensitive additive ones, take `confirm`:
  - Called with `confirm: false`, they return a **dry-run preview** of exactly what would change.
  - Called again with `confirm: true`, they execute.
  - **Always show the person the preview and get agreement before confirming.**
  - Gated: deletes, stopping a service, deploying, initialising a stack, clearing the temp cache,
    container stop/kill, running a command in a container, replacing an existing file over SFTP,
    nameserver and domain-contact changes, inviting someone to a service, and
    revoking access.
- **Overwrites are not gated, but are real.** These change live configuration with no preview, so state
  the change before making it:
  - setting env vars, a healthcheck or the deploy delay;
  - updating DNS records or toggling the Cloudflare proxy;
  - zone settings and bot management;
  - resetting SFTP or postbox passwords;
  - closing a ticket.
- **Failures come back as data.** A refused or failed operation returns
  `{ "succeeded": false, "outcome": "...", "message": "..." }`, where `outcome` is one of `NotFound`,
  `Invalid`, `NotOwner`, `NotManaged`, `LimitReached`, `Forbidden` or `Failed`. Relay the message
  plainly. A `succeeded:false` is not a success.
- **Never invent.** If a tool reports an error, an empty result or "not available", say so. Don't
  fabricate a service, a status, an image tag, a DNS record or a ticket to fill a gap.
- **Secrets are the member's own.** Some tools return the member's credentials: SFTP and SMTP details,
  freshly generated postbox passwords, SolidCP logins, database connection details, and the deploy
  webhook URL. Show them only to the person, and never write them into code or commits unless asked.
  Generated passwords are shown **once**.

## Finding things

Start from `services_list_my_services`, which gives the member's hosting services and their
**`serviceKey`**, the id almost every tool takes. From there:

- **Docker environments:** `docker_list_services` gives each environment's **`dockerServiceKey`**.
  Containers are addressed by the container id and node from `docker_list_containers`.
- **Cloudflare:**
  - A service's hostnames give a **`hostnameId`**. The DNS, zone-settings and bot tools need a
    **full zone** of the member's own, and refuse a custom hostname.
  - Custom hostnames are addressed by their Cloudflare id, from `cloudflare_hostnames_list`.
- **Registered domains:** `domain_list` gives each **`domainKey`**. Domain DNS works only when the
  domain has a linked Cloudflare zone.
- **Email domains:** addressed by name, from `email_list_domains`.
- **Tickets and licences:** belong to the member, not to a service (`tickets_list`, `licences_list`).

Tool names follow `<area>_<verb>[_<noun>]`. The tools are discoverable, so list them and call what the
server actually offers.

## What it exposes

| Area | Tools |
|------|-------|
| Services | `services_list_my_services`, `server_info` |
| Docker services | `docker_list_services`, `docker_get_service` (status, image, replicas, healthcheck, deploy delay), `docker_get_deploy_webhook`, `docker_get_environment_variables`, `docker_set_environment_variable`, `docker_remove_environment_variable`*, `docker_start_service`, `docker_stop_service`*, `docker_deploy_latest_image`*, `docker_initialize_stack`*, `docker_set_healthcheck`, `docker_set_deployment_delay`, `docker_clear_media_cache`, `docker_clear_temp_cache`* |
| Containers | `docker_list_containers`, `docker_get_container_stats`, `docker_get_container_logs`, `docker_container_action` (start/restart/stop/kill; all but start are gated), `docker_exec_command`*, `docker_read_container_file` |
| Registries | `docker_list_registries`, `docker_get_registry`, `docker_create_registry`, `docker_update_registry_credentials`, `docker_delete_registry`*, `docker_use_registry_for_service` |
| Cloudflare DNS | `cloudflare_dns_list_records`, `cloudflare_dns_get_record`, `cloudflare_dns_create_record`, `cloudflare_dns_update_record`, `cloudflare_dns_set_proxy`, `cloudflare_dns_delete_record`* |
| Cloudflare zone | `cloudflare_zone_get_settings`, `cloudflare_zone_set_setting`, `cloudflare_bot_get`, `cloudflare_bot_update`, `cloudflare_cdn_purge`, `cloudflare_analytics_get` |
| Custom hostnames | `cloudflare_hostnames_list`, `cloudflare_hostname_get` (the DNS and validation records to add), `cloudflare_hostname_limits`, `cloudflare_hostname_zones`, `cloudflare_hostname_create`, `cloudflare_hostname_validate`, `cloudflare_hostname_delete`* |
| Databases | `databases_list`, `databases_get`, `databases_get_onprem`, `databases_list_firewall_rules`, `databases_add_firewall_rule`, `databases_delete_firewall_rule`*, `databases_request_export`, `databases_list_exports`, `databases_get_export_download_url`, `databases_delete_export`* |
| SFTP | `sftp_list_users`, `sftp_get_user`, `sftp_create_user`, `sftp_reset_password`, `sftp_delete_user`*, `sftp_list_files`, `sftp_download_file`, `sftp_upload_file` (replacing an existing file is gated) |
| Email | `email_list_domains`, `email_get_domain_info` (DNS records to add), `email_verify_domain`, `email_list_postboxes`, `email_create_postbox`, `email_reset_postbox_password`, `email_delete_postbox`*, `email_list_events`, `email_get_stored_message`, `email_get_usage`, `smtp_list_accounts` |
| Registered domains | `domain_list`, `domain_check_availability`, `domain_get_info`, `domain_get_nameservers`, `domain_set_nameservers`*, `domain_get_contacts`, `domain_set_contacts`*, `domain_dns_list_records`, `domain_dns_get_record`, `domain_dns_create_record`, `domain_dns_update_record`, `domain_dns_set_proxy`, `domain_dns_delete_record`* |
| Sharing a service | `service_access_list`, `service_access_invite`*, `service_access_resend_invitation`, `service_access_revoke_invitation`*, `service_access_revoke`*. All are **owner only**; a delegated member gets `NotOwner`. |
| Support | `tickets_list`, `tickets_get`, `tickets_list_departments`, `tickets_list_priorities`, `tickets_create`, `tickets_reply`, `tickets_close` |
| Licences | `licences_list`, `licences_get` |
| IIS (Windows) | `iis_get_service` (temporary URL and SolidCP login; the site itself is managed in SolidCP) |

\* confirm-gated.

## Common flows

**Wire up and deploy a GreenStack site**
1. **Find the environment.** Use `services_list_my_services`, then `docker_list_services`, and pick the
   `dockerServiceKey`.
2. **Registry.** Create a registry credential with `docker_create_registry` (a read-scoped token for
   any Portainer-supported registry), then point the service at it with
   `docker_use_registry_for_service`.
3. **Provision or deploy.**
   - A new environment: `docker_initialize_stack` with the **bare image name only**
     (e.g. `acme/site`, no registry host; GreenStack pulls `latest`). Poll `docker_get_service` for
     `initializationStatus`.
   - An existing environment: `docker_deploy_latest_image`.
4. **CI.** Hand the person `docker_get_deploy_webhook` to store as the CI `WEBHOOK_ENDPOINT` secret. It
   exists only after the first deploy, so the very first CI push is expected to fail at the webhook step.
5. **Verify.** Use `docker_get_service` and `docker_get_container_logs`, then fetch the site until it
   responds healthy.

**Debug a misbehaving site**
1. **Container logs first.** `docker_get_container_logs` shows stdout and stderr: startup failures, unhandled
   exceptions, crash loops.
2. **Umbraco's own logs.** Read `umbraco/Logs` with `docker_read_container_file` (no binary needed in the
   image), or with `sftp_list_files` and `sftp_download_file` from the site's persistent files.
3. **Exec last, and minimally.** `docker_exec_command` runs one command as an argv array with no shell.
   GreenStack images are minimal: there is often no `bash` and sometimes no `sh`, so try the binary
   directly (`["dotnet", "--info"]`). Use `useShell: true` only if the image has `/bin/sh`. A missing binary
   comes back as `Invalid` with "executable file not found". Gated; show the command first.

**Persistence: mounts vs the container.** `docker_exec_command` and `docker_read_container_file` return the
container's mounts. Only files under a mount survive a deploy or restart; anything written elsewhere in the
container is lost. To change a site's files for good, write them with `sftp_upload_file`, which reaches the
persistent volume, and prefer changing the app in source control and redeploying. Uploads and downloads are
capped at 1 MB; use an SFTP client for bigger files (`sftp_get_user` gives the connection details).

**Add a custom hostname**
1. Call `cloudflare_hostname_create` with the hostname and the `dockerServiceKey` to route it to.
2. Call `cloudflare_hostname_get` for the CNAME target and the validation records, and give them to the
   person to add at their DNS.
3. Call `cloudflare_hostname_validate` once they're in place.

**DNS change**
Read the records first (`cloudflare_dns_list_records` or `domain_dns_list_records`), show the exact
change, then create, update or delete. Deletes are gated.

**Email setup**
`email_get_domain_info` gives the DNS records to add. Once they're added, `email_verify_domain`
re-checks them. Then `email_create_postbox`; the password is shown once.

**Raise a ticket**
Get a `departmentId` from `tickets_list_departments`, and optionally pass the `serviceKey` it's about.
Then `tickets_create`. Attachments can only be added in the portal.

## What this cannot do

- **No admin surface.** It's not a backoffice or admin tool, only the member's own customer-portal
  operations.
- **No buying.** It takes no payment and can't buy hosting, domains or add-ons; use the storefront MCP.
- **Portal-only actions:**
  - the interactive container terminal (single commands work via `docker_exec_command`);
  - the StackCP (20i) single sign-on;
  - ticket attachment upload and download.
- **No app code.** It doesn't configure the Umbraco application code; pair it with the
  `greenstack-umbraco-setup` skill for that.

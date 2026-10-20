# UmbPanel + CI orchestration

How to drive the UmbPanel side. Prefer the **UmbPanel MCP** (server `umbpanel`, from the companion
`umbpanel-mcp@umbhost` plugin — install it alongside this one; its `using-umbpanel-mcp` skill covers
the tools in full). When the plugin isn't installed, a tool is missing, or the connection is
unavailable, fall back to the manual portal steps — the app-side configuration never needs the MCP.

## MCP tools (member-scoped; you act as the logged-in portal member)

The tool names below are the ones the UmbPanel MCP serves:

| Need | Tool |
|---|---|
| List the member's services | `services_list_my_services`, then `docker_list_services` for each environment's `dockerServiceKey` |
| Read an environment: replica count/topology, current image, status, healthcheck | `docker_get_service` |
| Region, preview URL (`*.umbpanel.io`), image tag/registry, mounts, disk usage vs quota | `docker_get_service_information` |
| The deploy **`webHookUrl`** for the CI `WEBHOOK_ENDPOINT` secret | `docker_get_deploy_webhook` |
| Registry config (name/username/token/org) | `docker_list_registries` / `docker_get_registry` / `docker_create_registry` / `docker_update_registry_credentials` / `docker_delete_registry`, and `docker_use_registry_for_service` to point the environment at one |
| First deploy of a new environment with the **bare image name** | `docker_initialize_stack` |
| Deploy the latest image | `docker_deploy_latest_image` |
| Container status, logs and stats | `docker_list_containers` / `docker_get_container_logs` / `docker_get_container_stats` |
| Is the platform itself having an incident? | `platform_status` |

Replica count is **not self-serve**: the portal sends a sales ticket. Raise one with `tickets_create` (sales
department, from `tickets_list_departments`, with the `serviceKey`) rather than looking for a scale tool.

Provisioning/destructive tools are **confirm-gated**: called with `confirm: false` they return a
dry-run **preview**; call again with `confirm: true` to execute. Show the preview before confirming.

If the connection returns 401 / is dormant (member OAuth is enabled server-side at go-live), or a
tool is missing, present the equivalent manual portal step instead and continue.

## Manual portal steps (www.umbpanel.io → the service)

1. **Registry credential:** add the registry with a read-scoped token. GHCR: a classic PAT with
   `read:packages`; Docker Hub: an access token; ACR/ECR: the registry's service principal/keys.
   Fields: Name, Username (the registry account, **not** an email), Access Token, Organisation (if any).
2. **Image name:** in *Manage Websites*, enter the **bare image name only** — no registry host,
   username, or org (e.g. `my-site`, never `ghcr.io/acme/my-site`). GreenStack pulls `latest`.
3. **Replicas:** 1 = single, >1 = load-balanced. The count is changed by UmbHost on request (the
   portal's *Add more replicas* raises a sales ticket), within the package limit.
4. **Webhook secret (chicken-and-egg):** the `WEBHOOK_ENDPOINT` does not exist until after the
   first deploy, so the first CI push is **expected to fail at the webhook step**.

## Provider / registry matrix

Repository/CI provider and Docker registry are whatever the customer uses — nothing assumes
GitHub/GHCR. Pick the matching CI sample from `github.com/UmbHost/GreenStack.CICD.Samples`:

- **GitHub Actions:** `GitHub/main.yml` (GHCR), `GitHub/dockerhub.yml`, `acr.yml`, `aws-ecr.yml`,
  `gitlab-cr.yml`, `proget.yml`, `quay.yml`.
- **Azure DevOps:** the matching file under `AzureDevOps/`.
- Substitute the account/image placeholders (`GITHUB_ACCOUNT_NAME/DOCKER_IMAGE_NAME`) for the chosen
  provider+registry. The pipeline builds the image, pushes it, then `POST`s the deploy webhook.

## The deploy sequence (first time)

1. Commit the app changes, Dockerfile, `.dockerignore`, and the CI pipeline.
2. Push to the deploy branch → the image builds and pushes → the **webhook step fails** (expected).
3. Read the deploy webhook URL from UmbPanel (`docker_get_deploy_webhook` → `webHookUrl`, or the
   portal's deploy section). Treat it as secret-ish (the endpoint accepts an unauthenticated POST).
4. Store it as a CI secret/variable in the provider (e.g. `gh secret set WEBHOOK_ENDPOINT`, or an
   Azure DevOps pipeline secret) → re-run the pipeline. Now the deploy triggers.
5. **Verify:** GET the preview URL (`https://{service}.umbpanel.io`; `docker_get_service_information`
   gives the exact one) until it responds healthy. On failure, check `platform_status` for a platform
   incident, read `docker_get_container_logs`, then consult the health-check / "HTTPS is required" /
   casing / `App_Plugins` troubleshooting in the KB.

## Graceful degradation summary

- No MCP / OAuth dormant → do all app-side config, emit the exact manual portal steps.
- No registry credentials → stop at Dockerfile + CI pipeline + push instructions.
- Verification needs no MCP — it is a plain HTTPS GET of the temp URL.

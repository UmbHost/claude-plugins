# UmbPanel + CI orchestration

How to drive the UmbPanel side. Prefer the **UmbPanel MCP** (server `umbpanel`, from the companion
`umbpanel-mcp@umbhost` plugin — install it alongside this one; its `using-umbpanel-mcp` skill covers
the tools in full). When the plugin isn't installed, a tool is missing, or the connection is
unavailable, fall back to the manual portal steps — the app-side configuration never needs the MCP.

## MCP tools (member-scoped; you act as the logged-in portal member)

Provisional names (`<area>_<verb>_<noun>`), as the UmbPanel MCP ships them:

| Need | Tool |
|---|---|
| List the member's services | `services_list_my_services` |
| Read a service: replica count/topology, current image, status, **and the deploy `webHookUrl`** | `docker_get_service` |
| Set replica count | `docker_scale_service` |
| Registry config (name/username/token/org) | `docker_list_registries` / `docker_get_registry` / `docker_create_registry` / `docker_update_registry` / `docker_delete_registry` |
| Set the bare image name ("Manage Websites") | `docker_set_service_image` |
| Deploy latest image | `docker_deploy_latest_image` |
| Deployment / container status | `docker_get_container_status` / `docker_list_containers` |

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
3. **Replicas:** set the count (within the package limit). 1 = single, >1 = load-balanced.
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
3. Read the deploy webhook URL from UmbPanel (`docker_get_service` → `webHookUrl`, or the portal's
   deploy section). Treat it as secret-ish (the endpoint accepts an unauthenticated POST).
4. Store it as a CI secret/variable in the provider (e.g. `gh secret set WEBHOOK_ENDPOINT`, or an
   Azure DevOps pipeline secret) → re-run the pipeline. Now the deploy triggers.
5. **Verify:** GET `https://{service}.umbpanel.io` until it responds healthy. On failure, consult the
   health-check / "HTTPS is required" / casing / `App_Plugins` troubleshooting in the KB.

## Graceful degradation summary

- No MCP / OAuth dormant → do all app-side config, emit the exact manual portal steps.
- No registry credentials → stop at Dockerfile + CI pipeline + push instructions.
- Verification needs no MCP — it is a plain HTTPS GET of the temp URL.

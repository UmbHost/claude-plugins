---
name: using-umbpanel-mcp
description: Use when managing a member's EXISTING UmbHost GreenStack hosting through UmbPanel — listing their services, reading a service's status / replica count / image / deploy webhook, setting the container registry and image, triggering a deploy, or managing the site's Cloudflare DNS. Drives the authenticated UmbPanel MCP as the signed-in member (OAuth). Not for buying hosting (that is the UmbHost storefront MCP) and not for configuring the Umbraco app itself (that is the greenstack-umbraco-setup skill).
---

# Using the UmbPanel MCP

[UmbPanel](https://www.umbpanel.io) is UmbHost's control panel for **GreenStack** hosting. Its MCP
server lets an agent manage the hosting a member already has — their services, container image and
registry, deploys, and Cloudflare DNS — **acting as that member**.

**Endpoint:** `https://www.umbpanel.io/mcp` (Streamable HTTP). **Authenticated** — member OAuth 2.1
(authorization code + PKCE); the server advertises its auth via
`https://www.umbpanel.io/.well-known/oauth-protected-resource`. The first call triggers a browser
sign-in as the member; every tool then runs **scoped to that member**.

This is the **management** surface. *Buying* hosting is a different, public MCP
(`https://umbhost.net/mcp`, the UmbHost storefront). *Configuring and deploying the Umbraco app
itself* (Dockerfile, `Program.cs`, load balancing, CI) is the `greenstack-umbraco-setup` skill — this
MCP is how that skill carries out the UmbPanel-side steps.

## The rules that matter most

- **You are the member.** Every tool is scoped by UmbPanel's ownership checks to the signed-in
  member's own services and zones — you cannot see or touch anyone else's. If a call returns 401 or
  the connection is dormant (member OAuth not yet enabled on the instance), **say so** and fall back
  to telling the person the manual portal step; never pretend an action happened.
- **Destructive, billing and provisioning tools are confirm-gated.** Called with `confirm: false`
  they return a **dry-run preview** of exactly what would change; call again with `confirm: true` to
  execute. **Always show the person the preview and get agreement before confirming.**
- **Never invent.** If a tool reports an error, an empty result, or "not available", relay that
  plainly. Do not fabricate a service, a status, an image tag, or a DNS record to fill a gap.

## What it exposes

MCP tools are discoverable — list them and call what the server actually offers; the surface is
expanding. Tool names follow `<area>_<verb>_<noun>`. The families:

| Area | What you can do |
|------|-----------------|
| Services | List the member's services; read one service's status, replica count/topology, current image, and the **deploy `webHookUrl`**. |
| Deploys & scaling | Set the bare image name, set the replica count (1 = single, >1 = load-balanced), and trigger a deploy of the latest image. |
| Registries | List/read/create/update/delete the container-registry credential used to pull the image. |
| Containers | Read deployment and container/runtime status. |
| Cloudflare DNS | Read and create DNS records for the member's zone; deletes are confirm-gated. |

## The flow (typical: wire up and deploy a site)

1. **Find the service.** List the member's services, then read the target service — note its replica
   count (topology) and grab the deploy **`webHookUrl`** (it only exists after the first deploy).
2. **Registry + image.** Create the registry credential (any Portainer-supported registry, a
   read-scoped token), then set the **bare image name only** (no registry host/org — GreenStack pulls
   `latest`).
3. **Scale** to the intended replica count if it differs.
4. **Deploy.** Trigger the latest-image deploy, or hand the person the `webHookUrl` to wire into their
   CI as the `WEBHOOK_ENDPOINT` secret (the first CI push is expected to fail at the webhook step —
   that is where the URL comes from).
5. **Verify** by fetching `https://{service}.umbpanel.io` until it responds healthy.

For DNS, read the zone's records first, show the change, then create (or confirm-gated delete).

## What this cannot do

It is not a backoffice/admin surface and exposes no admin tooling — only the member's own
customer-portal operations. It takes no payment and cannot buy hosting (that is the storefront MCP).
It does not configure the Umbraco application code — pair it with the `greenstack-umbraco-setup`
skill for that.

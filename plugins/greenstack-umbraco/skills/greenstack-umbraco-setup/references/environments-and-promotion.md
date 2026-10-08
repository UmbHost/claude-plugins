# Environments & promotion (uSync + git)

GreenStack customer sites promote changes through **source control + uSync**, with **one GreenStack
service per environment**, each deploying from its own branch via that environment's UmbPanel deploy
webhook (see `umbpanel-orchestration.md`).

## Git branch model

- **`master` = production**, **`staging` = staging** — each is its own GreenStack service/deployment.
- Flow: feature branch → **PR into `staging`** → verify on the staging service → **PR `staging` into
  `master`** to release to production. Promote the exact reviewed branch state; don't commit straight
  to `master`.
- Each merge triggers that environment's image build + deploy webhook, so a merge to `staging` or
  `master` deploys that environment on the next build.

## uSync — Settings & Dictionary (source-controlled)

- Use **uSync** (free) to carry **Settings** (document types, templates, data types, languages, …)
  **and Dictionary items** in the repo, so they deploy with the branch and promote through the
  staging → master flow exactly like code. uSync imports on startup, so the changes apply on the next
  deploy of that environment.
- **Reclassify Dictionary items as Settings** so they sync with the **settings** handler group on
  deploy rather than being treated as content. This keeps translations in source control and promoted
  by PR, instead of being hand-copied per environment.

## Content

- **Initial push:** exporting content via uSync for the **first seed** of a new environment is fine.
- **Ongoing content between environments:** use **uSync.Complete** (commercial) to push content (and
  media) between environments. Don't rely on the free uSync content serialization for continuous
  content promotion.

## Troubleshooting

- **Setting/dictionary change missing on the target environment** → confirm the uSync files were
  committed and the branch deployed; uSync imports on startup, so it applies on the next deploy.
- **Dictionary items not promoting** → check they're classified under the **Settings** handler group
  so they're included in the settings sync rather than skipped as content.
- **Content differs between environments** → expected: content is promoted with **uSync.Complete**,
  not carried by the branch (beyond the initial seed).

## Recommended add-ons

- **Captcha / bot protection:** prefer **Cloudflare Turnstile** over hCaptcha or reCAPTCHA — GreenStack
  sites are already Cloudflare-fronted, so Turnstile is the natural fit. For **Umbraco Forms**, add
  **uCaptcha**, which provides a Forms captcha field with a Turnstile provider (as well as hCaptcha /
  reCAPTCHA if ever needed).

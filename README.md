# dokploy-extras

Custom Dokploy template registry for TheMusicDev. Live registry:
`https://themusicdev.github.io/dokploy-extras`

## Templates

| Template | Version | Image (pinned) | Port | Tags | Notes |
|---|---|---|---|---|---|
| [bugsink](blueprints/bugsink/) | 1.1.0 | `bugsink/bugsink:2.6.3` + `postgres:17-alpine` | 8000 | monitoring, error-tracking | Sentry-SDK compatible error tracker. `POSTGRES_PASSWORD`/`SECRET_KEY` auto-generated; requires `BASE_URL`/`ALLOWED_HOSTS` set to your real domain, initial login via `CREATE_SUPERUSER` |
| [meilisearch](blueprints/meilisearch/) | 1.1.0 | `getmeili/meilisearch:v1.57.0` | 7700 | search | `MEILI_MASTER_KEY` auto-generated (32 chars). Mini dashboard only in `MEILI_ENV=development` (dev mode doesn't enforce the master key — don't run on a public domain) |
| [umami](blueprints/umami/) | 1.1.0 | `ghcr.io/umami-software/umami:3.5.0` + `postgres:15-alpine` | 3000 | analytics | Privacy-focused web analytics. `APP_SECRET`/`POSTGRES_PASSWORD` auto-generated; `TWO_FACTOR_ENCRYPTION_KEY` needed before enabling 2FA |
| [hello-web](blueprints/hello-web/) | 1.0.0 | `nginx:alpine` | 80 | example | Minimal plumbing test |

Every template: **pinned image tags** (never `latest`), secrets auto-generated
via Dokploy's `${password:N}` helper (change only the human-choice fields the
deploy dialog flags), all documented env vars listed in `template.toml`
`[config.env]`, `json-file` log rotation (10 MB × 3) on every service,
official product logos.

## Use a template

In Dokploy:

1. Project → **Create Service** → **Template**
2. **Base URL**: `https://themusicdev.github.io/dokploy-extras`
3. Pick a template, review the env vars the deploy dialog shows (all supported
   vars are pre-listed with defaults), fill the `change_me` / `replace_me`
   secrets, deploy.
4. After the first deploy, rename the auto-generated domain
   (e.g. `Project → <template> → Domains`) and set the app's own domain
   reference if it has one (Bugsink's `BASE_URL`/`ALLOWED_HOSTS`) —
   `Project → <template> → Environment` then redeploy.

Reference for the test instances we keep in the `template-test` project:
`test-bugsink.2719.fyi`, `test-meilisearch.2719.fyi`, `test-umami.2719.fyi`.
Also verified working on Dokploy v2.12+ (compose + template deploys).

## Add a template

Conventions live in [CLAUDE.md](CLAUDE.md#conventions) — pinned tags, full env
var listing, log rotation, and proper logos are enforced for new templates.

Mechanics:

1. `blueprints/<id>/` with `docker-compose.yml`, `template.toml`, `logo.svg`
   (or `logo.png` when upstream only ships raster)
2. Add an entry to `meta.json` — `id` must match the folder name
3. Push; GitHub Pages serves it via
   [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml)

## Hosting

GitHub Pages via Actions workflow on push to `main`. Root of the repo is the
registry Dokploy reads (`meta.json` + `blueprints/<id>/`). `.nojekyll` included.
Pages was first enabled 2026-11-17 — before that the registry URL 404'd.

## Gotchas (Dokploy)

Short version of lessons learned — full list in
[CLAUDE.md](CLAUDE.md#gotchas-learned):

- `compose-deployTemplate` only *creates* the service; run deploy after.
- Redeploys are no-ops unless the compose file content changed (env-only
  changes don't count).
- Empty string != unset through Dokploy's env file — apps crash on
  empty-value flags (meilisearch).

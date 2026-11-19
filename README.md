# dokploy-extras

Custom Dokploy template registry for TheMusicDev. Live registry:
`https://themusicdev.github.io/dokploy-extras`

## Templates

| Template | Version | Image (pinned) | Port | Tags | Notes |
|---|---|---|---|---|---|
| [bugsink](blueprints/bugsink/) | 1.2.1 | `bugsink/bugsink:2.6.3` + `postgres:17-alpine` | 8000 | monitoring, error-tracking | Sentry-SDK compatible error tracker. `POSTGRES_PASSWORD`/`SECRET_KEY` auto-generated; requires `BASE_URL`/`ALLOWED_HOSTS` set to your real domain, initial login via `CREATE_SUPERUSER`; behind Cloudflare Flexible see the [post-deploy step](#bugsink-required-post-deploy-step-csrf-403-on-login) |
| [fider](blueprints/fider/) | 1.0.0 | `getfider/fider:v0.40.0` + `postgres:17-alpine` | 3000 | feedback, community | Customer feedback portal. `JWT_SECRET`/`POSTGRES_PASSWORD` auto-generated; **login is by emailed code, so set a real `EMAIL_SMTP_*` (or Mailgun/SES/OAuth) before first sign-in** |
| [posthog](blueprints/posthog/) | 1.0.1 | `posthog/posthog` @ `1e7b36e` + 14 other pinned images | 80 | analytics, feature-flags, session-replay | Trimmed hobby stack: Django, Celery, Node ingestion, ClickHouse, Redpanda, Postgres, Redis, MinIO. **Needs 8 GB+ RAM**, first boot 5–10 min on a fast CPU (~45 min on an old Xeon: thousands of DB migrations, the domain 502s until done). ClickHouse pinned to 26.3 LTS so it runs on CPUs without AVX2 (26.6+ needs it). Pinned to the 2026-07-14 snapshot, see [CLAUDE.md](CLAUDE.md#gotchas-learned). `POSTHOG_SITE_URL` defaults to `https://<domain>` (what the browser sees behind Cloudflare / Let's Encrypt); plain-http-only domains must set `http://` + `POSTHOG_SECURE_COOKIES=false` |
| [meilisearch](blueprints/meilisearch/) | 1.1.0 | `getmeili/meilisearch:v1.57.0` | 7700 | search | `MEILI_MASTER_KEY` auto-generated (32 chars). Mini dashboard only in `MEILI_ENV=development` (dev mode doesn't enforce the master key — don't run on a public domain) |
| [umami](blueprints/umami/) | 1.1.1 | `ghcr.io/umami-software/umami:3.5.0` + `postgres:15-alpine` | 3000 | analytics | Privacy-focused web analytics. `APP_SECRET`/`POSTGRES_PASSWORD` auto-generated; `TWO_FACTOR_ENCRYPTION_KEY` needed before enabling 2FA |
| [hello-web](blueprints/hello-web/) | 1.0.0 | `nginx:alpine` | 80 | example | Minimal plumbing test |

Every template: **pinned image tags** (never `latest`), secrets auto-generated
via Dokploy's `${password:N}` helper (change only the human-choice fields in the
Environment tab), all documented env vars listed in `template.toml`
`[config.env]`, `json-file` log rotation (10 MB × 3) on every service,
official product logos.

## Use a template

In Dokploy:

1. Project → **Create Service** → **Template**
2. **Base URL**: `https://themusicdev.github.io/dokploy-extras`
3. Pick a template and create it. The dialog only asks for a name (and
   server); it does **not** prompt for env vars. The domain is auto-generated
   (sslip.io) and secrets are generated for you.
4. Open `Project → <template> → Environment` and set the human-choice values
   (SMTP host/user/password, app URL such as Bugsink's `BASE_URL`/`ALLOWED_HOSTS`,
   Fider's `EMAIL_*`, PostHog's `POSTHOG_SITE_URL`). Env-only edits need
   **Deploy** (not Reload/Redeploy). Optionally rename the domain under
   `Project → <template> → Domains` first.

The `template-test` project is scratch space: test stacks get deleted after
each round (all cleaned up 2026-11-19). Also verified working on Dokploy v2.12+ (compose + template deploys).

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

## Bugsink: required post-deploy step (CSRF 403 on login)

Behind Cloudflare (Flexible SSL → Traefik `:80`) Traefik reports
`X-Forwarded-Proto: http`, so Bugsink rejects logins with
"CSRF verification failed". The template already defines the fix as a compose
label (`bugsink-https` middleware); you only need to attach it once:

**Service → Domains → edit the domain → Middlewares: `bugsink-https@docker`**,
then redeploy. Also set `BASE_URL` / `ALLOWED_HOSTS` to your real domain.

## Gotchas (Dokploy)

Short version of lessons learned — full list in
[CLAUDE.md](CLAUDE.md#gotchas-learned):

- `compose-deployTemplate` only *creates* the service; run deploy after.
- Redeploys are no-ops unless the compose file content changed (env-only
  changes don't count).
- Empty string != unset through Dokploy's env file — apps crash on
  empty-value flags (meilisearch).

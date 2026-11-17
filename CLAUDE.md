# CLAUDE.md — dokploy-extras

> Living document. Update it at the end of any session that changes the repo,
> learns a Dokploy template gotcha, or makes a decision about the roadmap.
> Keep it lean — facts that aren't in the code, not a file listing.

## What this is

Custom Dokploy **template registry** for TheMusicDev (git
`TheMusicDev/dokploy-extras`). Hosted on **GitHub Pages from `main` / repo
root** (`.nojekyll` present). Goal: a series of compose templates for our own
deployment needs, installable in Dokploy via project → Create Service →
Template → Base URL = the Pages URL (currently `https://themusicdev.github.io/dokploy-extras`).

## Registry format (Dokploy contract)

- Every template lives in `blueprints/<id>/` and **must** contain:
  `docker-compose.yml`, `template.toml`, `logo.svg`.
- `meta.json` (repo root) is the registry index; each entry's `id` must equal
  the folder name. Fields: `id`, `name`, `version`, `description`, `logo`,
  `links`, `tags`.
- `template.toml`: `[variables]` maps variable names to values — `${domain}`
  is a special Dokploy-provided variable for the domain prompt. `[config]`
  defines domains (serviceName + port + host) and env.
- Compose files reference variables as `${VAR}`; Dokploy fills them at deploy
  time. Images are pinned to a specific tag, never `latest`.
- No build step, no CI. Push to `main` = live registry.

## Workflow

1. New template = new folder + `meta.json` entry in the same change.
2. Validate locally before push:
   - `meta.json` is valid JSON and every entry has a matching folder (no
     official CLI — eyeball it or a jq one-liner).
   - compose YAML parses (`docker compose -f <file> config -q`).
3. Test against the real instance: **Dokploy MCP servers are connected in
   Claude Code** (`dokploy-mcp` / `dokploy-2719`). Use
   `compose-deployTemplate` / `compose-previewTemplate` / `compose-processTemplate`
   to verify a template actually deploys before calling it done. Prefer the
   live check over "looks right".
4. Commits only when explicitly asked.

## Conventions

- Template `id`: short kebab-case (`hello-web`).
- `version` bumped (semver) when compose/template behavior changes.
- Tags in `meta.json` used for categorization (`example`, `database`, `app`, …).
- **Pinned tags only.** Never `latest` in `image:`. Pin the exact version
  (e.g. `bugsink/bugsink:2.6.3`, `getmeili/meilisearch:v1.57.0`,
  `ghcr.io/umami-software/umami:3.5.0`). Verify the tag exists on the registry
  before pinning (docker registry `/tags/list`, ghcr ones need
  `?n=1000` — the default caps at 100 and hides newer tags).
- **Every supported env var in `template.toml` `[config.env]`.** The deploy
  dialog is the user's documentation — list all documented config vars with
  sane defaults (empty = disabled) so nothing has to be guessed. Copy-env
  lists verbatim from the project's official docs/compose, not from memory.
- **Log rotation on every service.** Each compose service gets:
  ```yaml
  logging:
    driver: json-file
    options:
      max-size: "10m"
      max-file: "3"
  ```
- **Proper logos.** `logo.svg` (or `logo.png` when upstream only ships
  raster) downloaded from an official source — product site or GitHub repo,
  never invented. Record where it came from in the session notes.
- Don't map host `ports:` in compose — Dokploy Traefik routes via
  `[[config.domains]]` (serviceName + internal port).

## Roadmap / decisions

<!-- Update as decisions are made. Oldest context first, newest at top. -->

- 2026-11-17: Repo scaffolded with `hello-web` example to verify Pages +
  registry plumbing.
- 2026-11-17: Added `bugsink`, `meilisearch`, `umami`. All three live-tested
  on the real instance — project `template-test`, env `production`, all
  healthy: bugsink (migrations on postgres ✓, health/ready 200), meilisearch
  (v1.57.0, /health 200), umami (v3.5.0, migrations ✓, Next.js ready).
- 2026-11-17: Dokploy MCP server `dokploy-mcp` times out in sessions — use
  `dokploy-2719` tools.

## Gotchas learned

<!-- Add non-obvious Dokploy/template findings here with dates. -->

- (2026-11-17) ghcr.io `/tags/list` caps at 100 entries by default — pass
  `?n=1000` or newer tags vanish silently.
- (2026-11-17) Umami v3 images are unified: plain numeric tags
  (`3.5.0`) on ghcr.io/umami-software/umami, PostgreSQL is the runtime
  default (no `DATABASE_TYPE` needed). The old `postgresql-vX.Y` tags are
  v1/v2-era only.
- (2026-11-17) Bugsink docker image configures everything via env vars
  including `DATABASE_URL` (postgres/mysql) — SQLite in /data is the baked-in
  default we don't use.
- (2026-11-17) Bugsink only ships raster logos (16–64 px PNG); 64 px it is.
- (2026-11-17) GitHub Pages was never actually enabled until now — the
  registry URL 404'd and `compose-templates` returned `[]`. Pages now runs
  via the `.github/workflows/deploy.yml` Actions workflow (build_type=workflow).
- (2026-11-17) **Empty string ≠ unset through the Dokploy env file.**
  meilisearch crashes with `error: a value is required for '--import-snapshot'`
  when a value-taking flag env var is set to `""`. Rule: never list
  value-taking CLI-flag env vars (meili import/dump/snapshot restore flags) in
  compose; anything left empty in template.toml must be tolerated by the app.
- (2026-11-17) `compose-redeploy` on a `sourceType: raw` compose is a no-op —
  it redeploys the *stored* compose file, it does not re-fetch the registry
  template. Template fixes require `compose-update` (paste new composeFile) or
  deleting the compose service and re-running `compose-deployTemplate`.
- (2026-11-17) `compose-deployTemplate` only *creates* the compose service
  (status idle, no deployment). A separate `compose-deploy` call is needed to
  actually deploy.
- (2026-11-17) Dokploy auto-creates an http sslip.io domain per compose
  service from `[config.domains]` (e.g.
  `templatetest-meilisearch-…-70-23-166-50.sslip.io`, port = configured port).
- (2026-11-17) Apps with hostname allowlists (bugsink/Django `BASE_URL` →
  `ALLOWED_HOSTS`) return 400 on the final domain until `BASE_URL` is set to
  the real host via the compose `.env` + redeploy. Dokploy's auto sslip.io
  domain can't be known at template-authoring time — fix env after deploy.
- (2026-11-17) Changing a compose domain requires a `compose-redeploy` after
  `domain-update` for the Traefik route to appear (raw compose: same stored
  file, but regenerates the per-app Traefik dynamic config on each deploy).
- (2026-11-17) Test instances for templates: project `template-test`,
  domains `test-bugsink.2719.fyi` (Bugsink, http→CF Full→Traefik),
  `test-meilisearch.2719.fyi` (Meilisearch), `test-umami.2719.fyi` (Umami) —
  all no-HTTPS-at-origin, *.2719.fyi wildcards through Cloudflare.
- (2026-11-17) Logo sources: meilisearch (repo main/assets/logo.svg),
  umami (umami.is /safari-pinned-tab.svg), bugsink (site 64 px PNG).

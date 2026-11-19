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
- **Secrets via Dokploy helpers, not baked values.** `template.toml` supports
  `${password} / ${password:N}` (random password at deploy time), plus
  `${hash:N}`, `${uuid}`, `${base64:N}`, `${randomPort}`, `${email}`,
  `${username}`, `${timestamp}`, `${jwt:secret:payload}`, `${domain}`.
  Use `${password:N}` for every secret; loud `change_me` placeholders only
  for values a human must choose (e.g. Bugsink `CREATE_SUPERUSER` login,
  since a generated email/password would be unknowable to the user).
  Helpers resolve once when the env file is generated, so redeploys keep
  the same value.
- **Every supported env var in `template.toml` `[config.env]`.** The
  service's Environment tab is the user's documentation (the create dialog asks
  only name/server, never env vars) — list all documented config vars with
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
- 2026-11-19: Added `fider` 1.0.0 (v0.40.0; Dokploy upstream was 0.21.1) and
  `posthog` 1.0.0 (July-2026 snapshot, all images pinned; see Gotchas). Both
  live-tested in `template-test`, then pushed (178d041) and **retested through
  the live registry** via `compose-deployTemplate` + `compose-deploy`: fider
  `/_health` 200 / `/signup` 200, posthog (~62 min first boot) `/_health` 200,
  `/` 302, `/flags` 200. Registry deploys land with the sslip.io domain — rename
  via `domain-update` + `compose-deploy` first. All four fider/posthog test
  stacks deleted afterwards (volumes included). Upstream Dokploy templates for
  these two were behind/broken (MinIO image gone, `:master` floats, `web` name
  collision). Next: bump PostHog only if upstream hobby stabilises.

## Gotchas learned

- (2026-11-19) **No user prompts in templates.** `template.toml` has no
  mechanism to ask for values: domain is auto-generated (sslip.io), helpers
  generate secrets, everything else is a fixed default the user edits in the
  Environment tab, then **Deploy**. Domain-dependent defaults (e.g. Bugsink
  `BASE_URL`) are wrong until edited.
- (2026-11-19) **PostHog healthcheck `start_period` must outlast first-boot
  migrations.** `posthog-web` runs `./bin/migrate && exec ./bin/docker-server`,
  not listening on :8000 until migrations finish (35–62 min on old Xeons).
  Domain 502s meanwhile (Caddy connection refused). With `start_period: 600s`
  the container flipped to "unhealthy" after ~17 min (cosmetic, nothing
  restarts it); 1.0.1 uses 5400s.

- (2026-11-19) **Dokploy upstream `posthog` template is broken**: its
  `minio/minio:RELEASE.2025-04-22…` image 404s on Docker Hub (MinIO stopped
  publishing; quay.io needs auth) — deploys die with "pull access denied".
  Our blueprint swaps in `chrislusf/seaweedfs:4.29` as `objectstorage:19000`
  (same swap upstream's hobby compose made), one `posthog` S3 identity from the
  install's generated keys, no anonymous access. Smoke-tested locally: signed
  PUT/GET 200, unsigned/wrong key 403. Web/replay services must
  `depends_on: objectstorage: service_healthy` (identity is registered after
  start by a background loop; sentinel file drives the healthcheck).
- (2026-11-19) **PostHog `SITE_URL` scheme must be the browser's (https), not the
  origin's.** With `http://` behind Cloudflare, signup/login work (unauthenticated
  requests skip CSRF) but every logged-in POST 403s: "CSRF Failed: Origin checking
  failed - https://… does not match any trusted origins". No Traefik middleware
  needed (unlike Bugsink) — just `POSTHOG_SITE_URL=https://…` +
  `POSTHOG_SECURE_COOKIES=true` (template defaults since 1.0.0). Env-only changes
  need `compose-update env` then **`compose-deploy`** (not `compose-redeploy`).
- (2026-11-19) **PostHog live-test result (test-posthog.2719.fyi):** 20 containers
  healthy; signup 201; login 200; authenticated HogQL POST with https Origin OK;
  `/e/` capture 200 → event found in ClickHouse (capture → Kafka → ingestion-general
  → ClickHouse works); `/flags`, `/array/<token>/config`, `/api/surveys` 200;
  `/s/` replay-capture reachable (400 on an empty probe). First boot ≈ 45 min on
  this Ivy Bridge host (thousands of Django migrations) — not 5–10 min. Session
  replay recording itself and livestream/CDP were not exercised.
- (2026-11-19) **Service-name collisions on `dokploy-network`.** The domain-routed
  service (PostHog's Caddy `proxy`) joins Dokploy's shared network, and every
  other stack's services are reachable there under their compose service name.
  `reverse_proxy web:8000` therefore round-robined into other apps' `web`
  containers — Bugsink answered `test-posthog.2719.fyi` with "400 Bad Request:
  Host …" (the page title said Bugsink; Rust paths like `/flags` worked because
  `feature-flags` is unique). Rule: **any service another service dials by DNS
  name from a domain-routed container must have a unique name** (`posthog-web`,
  not `web`/`app`/`api`/`db`). Upstream Dokploy's PostHog template has this bug.
  Diagnose by reading the 400/404 body, not just the status code.
- (2026-11-19) **ClickHouse ≥ 26.6 needs AVX2 (x86-64-v3).** The Dokploy test
  server is an Intel Xeon E5-2697 v2 (Ivy Bridge: AVX yes, **AVX2 no** — read via
  `docker-readContainerFile` on `/proc/cpuinfo`, base64 in the result). Upstream
  PostHog's pinned `clickhouse-server:26.6.1.1193` crash-looped with exit 132
  "Illegal instruction", leaving `web` stuck "Created". Fix: image is now
  `${CLICKHOUSE_VERSION}`, default `26.3.66.4` (26.3 LTS, SSE4.2 baseline). Bump
  to 26.6+ only on AVX2 hosts. Same CPU limit applies to any future ClickHouse
  template. Check flags first: `grep -w avx2 /proc/cpuinfo`.
- (2026-11-19) **PostHog snapshot choice.** Self-host has no tagged releases and
  upstream hobby compose (HEAD) now needs a custom ClickHouse image
  (`ghcr.io/posthog/clickhouse-posthog`), SeaweedFS x2, personhog, valkey,
  etcd, new capture env (`CAPTURE_OUTPUT_*` replaced `CAPTURE_V1_SINK_*`) — i.e.
  re-porting the trimmed stack. We stayed on the 2026-07-14 shape
  (`posthog/posthog:1e7b36e…`) and pinned every image from that era:
  `posthog-node:sha-82ea668` (also hobby's `latest`; Docker Hub node repo has
  had no newer than sha-c280973 since 2026-07-20), ghcr `capture` /
  `feature-flags` / `hypercache-server` `sha-0c41485`, `livestream`
  `sha-e6a7738`. All are env vars in `template.toml` (`*_VERSION`) so bumping is
  one place. Upstream Dokploy template used floating `:master` for those four.
  Finding era tags: GitHub Packages API
  (`gh api /orgs/PostHog/packages/container/posthog%2F<img>/versions`, newest
  first, ~100/page, 100s of pages — binary-search on `updated_at`); ghcr
  `/tags/list` is lexicographic and useless for recency.
- (2026-11-19) PostHog docs env list: only list vars whose documented default
  is empty or sane; skip `None`-default ones (TRUSTED_PROXIES, STATSD_*,
  GITHUB_TOKEN...) — an empty string through the env file isn't `None`. Any
  unlisted var can be added to the compose `environment`.
- (2026-11-19) PostHog logomark: official `PostHog/posthog.com`
  `static/brand/posthog-logomark-padded.svg`, head recolored `#111`→`#fff`
  (dark Dokploy UI, same fix as umami). Fider: `etc/logo.png` (300×300) from
  `getfider/fider` — repo ships no SVG.
- (2026-11-19) **Fider** v0.40.0 (Dokploy upstream was on 0.21.1): all env vars
  copied from `app/pkg/env/env.go`. Logs users in by **emailed code** and
  panics at startup if `EMAIL_SMTP_HOST`/`PORT` are empty (smtp is inferred when
  `EMAIL` is empty), so the template ships a placeholder `smtp.example.com`
  — user must set real SMTP before first login. Dropped upstream's `mailhog`
  (no UI exposed, swallowed every mail). Empty-string string vars are fine on
  Fider; typed vars keep real defaults. Live-tested: `/` → 307 `/signup`
  (first-run wizard), both containers healthy, no http→https redirect loop
  behind Cloudflare Flexible.
- (2026-11-19) Testing without pushing: `compose-create` (raw) →
  `mounts-create` (file mounts, `mountPath: "/"`, `filePath` like the toml) →
  `compose-update` (`composeFile` + raw `env`) → `domain-create` →
  `compose-deploy`. Needs the full compose typed into the call — a registry
  push + `compose-deployTemplate` is cheaper when pushing is allowed.
- (2026-11-19) Server for tests has 135 GB RAM / 24 CPU — PostHog's 8 GB+
  requirement is no problem there.

<!-- Add non-obvious Dokploy/template findings here with dates. -->

- (2026-11-18) **Bugsink CSRF 403 behind Cloudflare Flexible — solved.** CF
  talks plain http to Traefik `:80`, Traefik stamps `X-Forwarded-Proto: http`,
  Django computes expected origin `http://…` but the browser sends
  `Origin: https://…` → "CSRF verification failed". Fix = headers middleware
  defined as a **Traefik label inside the compose service**
  (`traefik.http.middlewares.bugsink-https.headers.customrequestheaders.X-Forwarded-Proto=https`,
  now in the bugsink blueprint, v1.2.0) and attached on the domain as
  `bugsink-https@docker` (domain-update `middlewares`). Mirrors the user's
  older working instance `homelab-bugsink-b9xege`. Applied + verified on the
  test stack and on the user's `okf-bugsink.2719.fyi` (login POST 403 → 200).
- (2026-11-18) Did NOT work, don't retry: file-based middleware in
  `middlewares.yml` (router referencing it 404'd), Traefik
  `forwardedHeaders.trustedIPs`, `BEHIND_PLAIN_HTTP_PROXY`, origin
  HTTPS/letsencrypt (CF Flexible → 301 loop). **The user will not change the
  Cloudflare SSL mode** — other services depend on Flexible. Never ask again.
- (2026-11-18) `template.toml` can't attach a domain middleware, so after
  deploying bugsink from the registry the domain's Middleware must be set to
  `bugsink-https@docker` (documented in README; or use domain-update).
- (2026-11-18) Debug aid: `DEBUG_CSRF=True` + POST `/debug/csrf/` shows the
  headers Django sees. A bad-password login POST returning 200 (not 403)
  proves CSRF passes. Compose ids change if the user recreates a service —
  re-run `project-all` rather than trusting old ids.

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
- (2026-11-17) Bool env casing is intentionally per-app, don't "standardize":
  bugsink is case-insensitive (source: `conf_templates/docker.py.template`
  does `.lower() in ("true","1","yes")`), so `True`/`true`/`1` all work;
  meilisearch (clap) requires lowercase `true`/`false` and errors on `True`;
  umami flags are `"1"`. Casing varies BETWEEN apps, but must be uniform
  WITHIN one template (2026-11-19: bugsink had `True`/`False` mixed with
  `true`/`false` from copying the docs verbatim — now all lowercase, 1.2.1).
  Already-deployed instances keep their old env values; helpers/defaults only
  apply at creation. Bugsink `EMAIL_BACKEND` must stay empty (a bare `smtp`
  is not a valid Django backend path and breaks startup/sending); Django only
  authenticates SMTP when user AND password are both non-empty.
- (2026-11-17) Template.toml DOES support secret generation
  (`${password:N}`) — an earlier wrong claim here said it didn't. All three
  templates bumped to v1.1.0 using it. Don't trust remembered feature lists
  — check the official Dokploy/templates README.
- (2026-11-17) Duplicate TOML keys (a later edit appended a duplicated
  `MEILI_SNAPSHOT_DIR`) slip past `docker compose config` — always validate
  template.toml with a real TOML parser too.
- (2026-11-17) `compose.update` MCP `env` field is **raw env-file text** —
  sending base64 writes garbage and wipes every variable (all compose
  interpolations warn "not set"). Verified working raw.
- (2026-11-17) `compose-redeploy` no-ops (0.2s deploys) unless the stored
  compose file content changed — an env-only change won't redeploy. Force a
  real deploy by also sending a tweaked `composeFile` via compose.update.
- (2026-11-17) Domain routes are Traefik labels baked into the container at
  creation (`traefik.http.routers…Host(...)`), not a separate dynamic file.
  A crashlooping/unhealthy container behind the label makes the domain 404 —
  check container health first before blaming Traefik.
- (2026-11-17) Verified reachable (no-origin-HTTPS, via Cloudflare → Traefik
  :80): `https://test-bugsink.2719.fyi` ("Log in · Bugsink", route:
  /accounts/login/), `https://test-umami.2719.fyi` (Umami), and
  `https://test-meilisearch.2719.fyi/health` (API auth with master key ✓).
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
  umami (`src/assets/logo-white.svg` from umami-software/umami — the bowl
  mark, already white for dark UIs; 2026-11-19 replaced the earlier
  umami.is `safari-pinned-tab.svg` potrace blob, which rendered as a plain
  white disc), bugsink (site 64 px PNG).
- (2026-11-17) Meilisearch's mini dashboard UI only runs with
  `MEILI_ENV=development`; in production the root just returns
  `{"status":"Meilisearch is running"}`. Don't switch the public domain to
  dev — in development the master key is NOT enforced (open admin access).

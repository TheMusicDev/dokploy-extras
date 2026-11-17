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

## Roadmap / decisions

<!-- Update as decisions are made. Oldest context first, newest at top. -->

- 2026-11-17: Repo scaffolded with `hello-web` example to verify Pages +
  registry plumbing. Next: list the real templates we actually need.

## Gotchas learned

<!-- Add non-obvious Dokploy/template findings here with dates. -->

- (none yet)

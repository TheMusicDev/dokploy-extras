# dokploy-extras

Custom Dokploy template registry for TheMusicDev.

## Use it
In Dokploy: project → Create Service → Template → set **Base URL** to the hosted URL of this repo
(e.g. `https://themusicdev.github.io/dokploy-extras`). Dokploy reads `meta.json` and `blueprints/<id>/`.

## Add a template
1. `blueprints/<id>/` with `docker-compose.yml`, `template.toml`, `logo.svg`
2. Add an entry to `meta.json` (`id` must match the folder name)
3. Push; GitHub Pages serves it.

## Hosting
GitHub Pages: Settings → Pages → deploy from `main` / root. `.nojekyll` is included.

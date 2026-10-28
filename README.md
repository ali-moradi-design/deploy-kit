# deploy-kit

Reusable deploy pipelines and server templates for the demo sites on one VPS
(`<name>.ali-moradi-develop.com`). Nothing secret lives here: keys and
passwords stay in each repo's GitHub environment and in `.env` on the server.

| Kind | Workflow | Templates and checklist |
| --- | --- | --- |
| Static site (files served by Nginx) | `.github/workflows/deploy-static.yml` | `static/` |
| Dynamic app (Docker + own database) | coming next | `dynamic/` |

## Using a workflow

```yaml
jobs:
  deploy:
    uses: ali-moradi-design/deploy-kit/.github/workflows/deploy-static.yml@<full commit SHA> # v1
    with:
      site-url: https://cafe.ali-moradi-develop.com
      output-dir: out
```

- Pin the full commit SHA, not `main`: a change here then reaches a demo only when that demo updates the SHA
  (Dependabot does it with a PR, so the demo's CI checks the new version first).
- The deploy job runs in the caller's environment (default `production`) and reads `SSH_*` from there;
  the caller passes no secrets.
- The caller's `permissions` must allow at least `contents: read`.

## Server layout (same for every site)

- Static: `/srv/static/<name>`, key forced to `rrsync` for that folder.
- Dynamic: `/srv/apps/<name>/compose.yaml` + `.env`, port only on `127.0.0.1`.
- Nginx on the host: `/etc/nginx/sites-available/<name>` symlinked into `sites-enabled`, one certbot certificate per site.

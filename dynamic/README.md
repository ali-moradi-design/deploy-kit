# New dynamic app: checklist

Example values: name `shop-books`, domain `shop-books.ali-moradi-develop.com`, port `3101`.
Replace them everywhere. One name = one folder, one compose project, one key, one Nginx file, one certificate.
Take the next free port from the port table in the roadmap notes and add the app there.

## 1. Repo

- Dockerfiles that build production images (multi-stage, non-root user, `HEALTHCHECK` or a health route).
- `deploy/compose.yaml` from `compose.yaml` here, `deploy/nginx.conf` from `nginx.conf`, `deploy/.env.example`
  (names only, no values).
- `.github/workflows/docker.yml` from `caller.yml`: the images job builds `<name>-web:<sha>` (and `-api`) and
  pushes them to ghcr.io; the deploy job calls `deploy-app.yml`. After the first push make each package public
  (package → Settings → Change visibility) so the server pulls without logging in.

## 2. Folder, compose file and secrets on the server

```bash
sudo install -d -o deploy -g deploy -m 755 /srv/apps/shop-books
cd /srv/apps/shop-books
sudo -u deploy curl -fsSL -o compose.yaml https://raw.githubusercontent.com/ali-moradi-design/shop-books/main/deploy/compose.yaml
sudo -u deploy install -m 600 /dev/null .env
sudo -u deploy sh -c 'printf "IMAGE_TAG=none\nDB_PASSWORD=%s\nJWT_ACCESS_SECRET=%s\n" "$(openssl rand -hex 32)" "$(openssl rand -hex 32)" >> .env'
sudo -u deploy sed 's/=.*/=***/' .env          # names only, values hidden
sudo -u deploy docker compose config --quiet && echo OK
sudo -u deploy docker compose up -d --wait db  # only the database until the images exist
```

`IMAGE_TAG=none` is a placeholder; the first deploy writes the commit SHA.

## 3. Deploy key (PowerShell, on the PC)

```powershell
ssh-keygen -t ed25519 -C "github-actions-shop-books" -f $env:USERPROFILE\.ssh\gha_shop_books   # no passphrase
Get-Content $env:USERPROFILE\.ssh\gha_shop_books.pub | Set-Clipboard
```

On the server, add ONE line to `/home/deploy/.ssh/authorized_keys` (`sudo -u deploy nano ...`):

```text
command="/usr/local/bin/deploy-app shop-books",restrict ssh-ed25519 AAAA... github-actions-shop-books
```

The key can only run `deploy-app` for this app with a commit SHA (no shell, no other app). `deploy-app` must be
installed once (`server/README.md`).

## 4. GitHub environment

Repo → Settings → Environments → New environment `production`, deployment branches: only `main`.

- Variables: `SSH_HOST`, `SSH_PORT`, `SSH_USER` (same values as the other demos, kept in the private roadmap notes)
- Secret `SSH_PRIVATE_KEY`: `Get-Content -Raw $env:USERPROFILE\.ssh\gha_shop_books | Set-Clipboard`
- Secret `SSH_KNOWN_HOSTS`: `ssh vps "ssh-keyscan -p <SSH port> -t ed25519 <server IP> 2>/dev/null" | Set-Clipboard`

## 5. Nginx and certificate

Once for all apps (skip if `/etc/nginx/conf.d/connection-upgrade.conf` exists):

```bash
sudo curl -fsSL -o /etc/nginx/conf.d/connection-upgrade.conf \
  https://raw.githubusercontent.com/ali-moradi-design/deploy-kit/main/dynamic/connection-upgrade.conf
```

Per app (the repo's own `deploy/nginx.conf`, already filled):

```bash
sudo curl -fsSL -o /etc/nginx/sites-available/shop-books \
  https://raw.githubusercontent.com/ali-moradi-design/shop-books/main/deploy/nginx.conf
sudo ln -s /etc/nginx/sites-available/shop-books /etc/nginx/sites-enabled/shop-books
sudo nginx -t && sudo systemctl reload nginx     # 502 until the app runs: expected
sudo certbot certonly --nginx --dry-run -d shop-books.ali-moradi-develop.com
sudo certbot --nginx -d shop-books.ali-moradi-develop.com
```

## 6. First deploy and checks

- Merge into `main` (or Actions → Docker → Run workflow). The deploy job prints `deploy-app: shop-books none -> <sha>`.
- Seed once if the app needs data: `sudo -u deploy docker compose exec api <seed command>`.
- `docker compose ps` all healthy; `docker stats --no-stream` (memory under the limits); `free -h`.

## 7. Clean up and protect

- Delete the private key from the PC; GitHub holds it.
- Ruleset `protect-main`: PR required (0 approvals), required check `ci`, no force push or deletion.
- Nightly demo reset if the app has one: `sudo crontab -u deploy -e`, e.g.
  `30 3 * * * cd /srv/apps/shop-books && docker compose exec -T api <reset command> 2>&1 | logger -t shop-books-reset`
- Rollback by hand: `sudo -u deploy cp .env.previous .env && sudo -u deploy docker compose up -d --wait`
  (deploy-app already rolls back by itself when the new containers are not healthy).
- Old images are removed nightly by `cleanup-images` (it reads `IMAGE_TAG` from every `/srv/apps/*/.env`).

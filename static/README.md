# New static site: checklist

Example values: name `cafe`, domain `cafe.ali-moradi-develop.com`.
Replace them everywhere. One name = one folder, one key, one Nginx file, one certificate.

## 1. Repo

- `pnpm build` (or another command) writes a folder with `index.html` in it.
- Copy `caller.yml` to `.github/workflows/deploy.yml`, fill `site-url`, `output-dir`, `smoke-paths`
  and the deploy-kit commit SHA.

## 2. Folder on the server

```bash
sudo install -d -o deploy -g deploy -m 755 /srv/static/cafe
```

## 3. Deploy key (PowerShell, on the PC)

```powershell
ssh-keygen -t ed25519 -C "github-actions-cafe" -f $env:USERPROFILE\.ssh\gha_cafe   # no passphrase
Get-Content $env:USERPROFILE\.ssh\gha_cafe.pub | Set-Clipboard
```

On the server, add ONE line to `/home/deploy/.ssh/authorized_keys` (`sudo -u deploy nano ...`):

```text
command="/usr/bin/rrsync /srv/static/cafe",restrict ssh-ed25519 AAAA... github-actions-cafe
```

`rrsync` = this key can only rsync inside that folder (no shell, no Docker).

## 4. GitHub environment

Repo → Settings → Environments → New environment `production`, deployment branches: only `main`.

- Variables: `SSH_HOST`, `SSH_PORT`, `SSH_USER` (same values as the other demos, kept in the private roadmap notes)
- Secret `SSH_PRIVATE_KEY`: `Get-Content -Raw $env:USERPROFILE\.ssh\gha_cafe | Set-Clipboard`
- Secret `SSH_KNOWN_HOSTS` (run on the PC; keyscan runs on the server):
  `ssh vps "ssh-keyscan -p <SSH port> -t ed25519 <server IP> 2>/dev/null" | Set-Clipboard`

## 5. Nginx and certificate

Once for all static sites (skip if `/etc/nginx/snippets/static-security-headers.conf` exists):

```bash
sudo curl -fsSL -o /etc/nginx/snippets/static-security-headers.conf \
  https://raw.githubusercontent.com/ali-moradi-design/deploy-kit/main/static/security-headers.conf
```

Per site:

```bash
curl -fsSL https://raw.githubusercontent.com/ali-moradi-design/deploy-kit/main/static/nginx.conf \
  | sed -e 's/__NAME__/cafe/g' -e 's/__DOMAIN__/cafe.ali-moradi-develop.com/g' \
  | sudo tee /etc/nginx/sites-available/cafe > /dev/null
sudo ln -s /etc/nginx/sites-available/cafe /etc/nginx/sites-enabled/cafe
sudo nginx -t && sudo systemctl reload nginx
sudo certbot certonly --nginx --dry-run -d cafe.ali-moradi-develop.com
sudo certbot --nginx -d cafe.ali-moradi-develop.com
```

Vite site: change `/_next/static/` to `/assets/` and use the single-page `try_files` line (comments in the file).

## 6. First deploy and checks

- Actions → Deploy → Run workflow (or merge a PR into `main`).
- `curl -sI https://cafe.ali-moradi-develop.com` → 200 and the security headers; `curl -X POST ...` → 405.

## 7. Clean up and protect

- Delete the private key from the PC (`Remove-Item $env:USERPROFILE\.ssh\gha_cafe`); GitHub holds it.
- Ruleset `protect-main` on the repo: PR required (0 approvals), required check `ci`, no force push or deletion.
  Every merge deploys, so only green code may reach `main`.
- Add the site to the port/site table in the roadmap notes.

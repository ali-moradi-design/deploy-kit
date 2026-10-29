# Server scripts

Installed once on the VPS, owned by root, used by every dynamic app.

| Script | Path on the server | Used by |
| --- | --- | --- |
| `deploy-app` | `/usr/local/bin/deploy-app` | forced command of each app's deploy key |
| `cleanup-images` | `/usr/local/bin/cleanup-images` | nightly cron of `deploy` |

Install or update (read the file first, then):

```bash
curl -fsSL -o /tmp/deploy-app https://raw.githubusercontent.com/ali-moradi-design/deploy-kit/main/server/deploy-app
less /tmp/deploy-app
sudo install -m 755 /tmp/deploy-app /usr/local/bin/deploy-app && rm /tmp/deploy-app
```

Test without a deploy: as `deploy`, a wrong tag must be refused and change nothing:

```bash
sudo -u deploy env SSH_ORIGINAL_COMMAND='not-a-sha' /usr/local/bin/deploy-app leather-shop; echo "exit $?"   # exit 2
```

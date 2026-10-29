# Database blocks

Each app has its OWN database container: private `backend` network, no published ports, named volume,
healthcheck, memory limit. Swap the `db` service in `compose.yaml` for one of these.

## Postgres (default in compose.yaml)

`postgres:<exact version>-alpine`, `mem_limit: 256m`, healthcheck `pg_isready`.

## MySQL

```yaml
  db:
    image: mysql:8.4.6 # pin the exact version
    command: ["--innodb-buffer-pool-size=64M", "--performance-schema=OFF", "--skip-name-resolve"]
    environment:
      MYSQL_DATABASE: app
      MYSQL_USER: app
      MYSQL_PASSWORD: ${DB_PASSWORD:?DB_PASSWORD is not set}
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD:?DB_ROOT_PASSWORD is not set}
    volumes:
      - db-data:/var/lib/mysql
    networks: [backend]
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -u app -p\"$$MYSQL_PASSWORD\" --silent"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s
    mem_limit: 384m
    restart: unless-stopped
```

URL for the app: `mysql://app:${DB_PASSWORD}@db:3306/app`.

## MongoDB

```yaml
  db:
    image: mongo:8.2.6 # 8.3 refuses to start on Linux 6.19+ (SERVER-121912)
    command: ["--wiredTigerCacheSizeGB", "0.25", "--auth"]
    environment:
      MONGO_INITDB_ROOT_USERNAME: app
      MONGO_INITDB_ROOT_PASSWORD: ${DB_PASSWORD:?DB_PASSWORD is not set}
    volumes:
      - db-data:/data/db
    networks: [backend]
    healthcheck:
      test: ["CMD", "mongosh", "--quiet", "--eval", "db.adminCommand('ping')"]
      interval: 15s
      timeout: 5s
      retries: 5
    mem_limit: 512m
    restart: unless-stopped
```

URL for the app: `mongodb://app:${DB_PASSWORD}@db:27017/app?authSource=admin`.

## Passwords

Generate with `openssl rand -hex 32` (hex only: base64 `/` and `+` break connection URLs).
The database stores the password only on the FIRST start of an empty volume; changing `.env` later does not change it.

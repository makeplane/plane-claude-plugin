# Plane Community with Docker Compose (`setup.sh`)

The supported path is `setup.sh` from the latest `makeplane/plane` GitHub release (v1.4.3 as of
2026-11). Images are multi-arch (amd64 and arm64), so no platform flags are needed.

## 1. Download and review

```bash
mkdir -p plane-selfhost && cd plane-selfhost
curl -fsSL -o setup.sh https://github.com/makeplane/plane/releases/latest/download/setup.sh
chmod +x setup.sh
```

Summarize the script for the user and get confirmation before running it.

`setup.sh` takes the action as its first argument, so you don't need to drive the menu:
`install`, `start`, `stop`, `restart`, `upgrade`, `logs <service>` and `backup`. The menu has
the same actions: 1 Install · 2 Start · 3 Stop · 4 Restart · 5 Upgrade · 6 View Logs ·
7 Backup Data · 8 Exit.

## 2. Install (downloads only, starts nothing)

```bash
./setup.sh install
```

This creates `plane-app/docker-compose.yaml` and `plane-app/plane.env`, writes the resolved
release tag into `APP_RELEASE`, and pulls the images.

## 3. Configure `plane-app/plane.env` before the first start (never print it)

```bash
cd plane-app; F=plane.env; chmod 600 $F
setkey() { if grep -q "^$1=" "$3"; then sed -i.bak "s|^$1=.*|$1=$2|" "$3" && rm -f "$3.bak"; else echo "$1 not in $3 (skipped)"; fi; }
DOMAIN=plane.example.com
setkey APP_DOMAIN "$DOMAIN" $F
setkey WEB_URL "https://$DOMAIN" $F                 # or http://<host>[:port]
setkey CORS_ALLOWED_ORIGINS "https://$DOMAIN" $F    # same as WEB_URL
setkey SECRET_KEY "$(openssl rand -hex 32)" $F      # never rotate later
setkey LIVE_SERVER_SECRET_KEY "$(openssl rand -hex 32)" $F
PG=$(openssl rand -hex 16); MQ=$(openssl rand -hex 16); S3=$(openssl rand -hex 16)
setkey POSTGRES_PASSWORD "$PG" $F
setkey DATABASE_URL "postgresql://plane:$PG@plane-db/plane" $F
setkey RABBITMQ_PASSWORD "$MQ" $F
setkey AMQP_URL "amqp://plane:$MQ@plane-mq:5672/plane" $F
setkey AWS_SECRET_ACCESS_KEY "$S3" $F               # also the bundled MinIO root password
unset PG MQ S3
```

- If `APP_DOMAIN` is the only thing set, `WEB_URL` defaults to `http://${APP_DOMAIN}`. Set both
  URLs explicitly for https or a non-80 port (`LISTEN_HTTP_PORT` / `LISTEN_HTTPS_PORT`).
- The compose file hardcodes `plane:plane` in the default `DATABASE_URL` and `AMQP_URL`. That's
  why each password is set together with its URL. Check `POSTGRES_USER`, `POSTGRES_DB`,
  `RABBITMQ_USER` and `RABBITMQ_VHOST` in the file, and adjust the URLs if they differ from
  `plane` / `plane`.
- Passwords only take effect when the volumes are first created, so set them before the first
  start.

**TLS with the bundled Caddy** (Let's Encrypt, public DNS to this host, 80/443 open):
`SITE_ADDRESS=<domain>` and `CERT_EMAIL=email you@example.com`. The `email ` prefix is
required: the value is inserted as a raw Caddy option. For a DNS challenge, add
`CERT_ACME_DNS=acme_dns <provider> <token>` (the user types the token). Keep the URLs on https.

**External services**: full URLs only, using `DATABASE_URL`, `REDIS_URL` and `AMQP_URL`.
`PGHOST`, `REDIS_HOST` and similar keys have no effect. For S3, set `USE_MINIO=0` plus
`AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_S3_ENDPOINT_URL` and
`AWS_S3_BUCKET_NAME`. The bundled containers keep running unless you remove them and their
`depends_on` entries from `docker-compose.yaml`. Check with `docker compose config -q`.

**Keys that do nothing in v1.4.3**, so don't set them for the user: `TRUSTED_PROXIES`,
`GUNICORN_WORKERS`, `DOCKER_PLATFORM`, `DOCKERHUB_USER` and `AUTHENTICATION_RATE_LIMIT`. There are
no SMTP keys either. Outbound email is configured in God Mode.

## 4. Start

```bash
cd .. && ./setup.sh start
```

It waits for the migrator to succeed, then polls the API for up to 5 minutes and prints the URL.
Then follow `verification.md`.

## Lifecycle

| Action | Command | Notes |
| --- | --- | --- |
| Stop / restart | `./setup.sh stop` · `./setup.sh restart` | Restart applies env changes |
| Logs | `./setup.sh logs api` | Services: web, space, api, worker, beat-worker, migrator, proxy, redis, postgres, minio, rabbitmq |
| Back up | `./setup.sh backup` | Containers must be running. Writes `plane-app/backup/<timestamp>/` with pgdata, redisdata, uploads and rabbitmq_data archives. Keep `plane.env` with it |
| Restore | `curl -fsSL -o restore.sh https://github.com/makeplane/plane/releases/latest/download/restore.sh && chmod +x restore.sh && ./restore.sh <backup folder>` | Needs `jq`. Plane must have been installed and started once, then stopped. It replaces the volumes. Run `./setup.sh start` afterwards |
| Upgrade | `./setup.sh upgrade` | Asks y/N even with the argument, so the user answers it. It stops services, replaces `plane.env` with the new template and re-applies the old values (the old file is kept as `plane.env.bak` and under `archive/`), then **doesn't start**. Check the merged `plane.env` for new keys, then `./setup.sh start` |

Back up before every upgrade. `PLANE_DEPLOYMENT.md` goes in `plane-selfhost/`.

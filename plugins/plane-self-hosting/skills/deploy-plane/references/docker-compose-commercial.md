# Plane Commercial with Docker Compose (manual)

Use this path when the user wants to own the compose files. For a managed lifecycle on Linux,
`prime-cli.md` is simpler, and it also drops the bundled containers when you point at external
services.

## 1. Download the release files

Release files are version-pinned. Resolve the version first: the user's pin, or the latest from
the releases page <https://plane.so/changelog?category=self-hosted> (v4.0.1 as of 2026-11).

```bash
VER=v4.0.1
mkdir -p plane-ee && cd plane-ee
curl -fsSLO "https://prime.plane.so/releases/${VER}/docker-compose.yml"
curl -fsSLO "https://prime.plane.so/releases/${VER}/variables.env"
grep -q '^services:' docker-compose.yml || echo "docker-compose.yml is NOT a compose file"
grep -q '^APP_RELEASE_VERSION=' variables.env || echo "variables.env is NOT the Plane env file"
chmod 600 variables.env
```

Equivalent zip: `curl -fsSL "https://prime.plane.so/api/v2/setup/?version=${VER}" -o plane.zip`.

Don't use the older `https://prime.plane.so/deployment/assets/<ver>/…` path. It returns HTTP 200
with an HTML portal page, not the files. That's why the script above checks the content.

The compose file has **no `env_file:`**. A key only reaches the containers if
`docker-compose.yml` interpolates it. Before adding a key the user mentions, confirm it's there:
`grep -n 'KEY' docker-compose.yml`.

## 2. Edit `variables.env` (never print it)

Use this helper. It changes an existing key, never adds one, and works with both GNU and BSD
`sed`. It's only for generated hex values and plain values like a domain. Values with `|`, `&`
or `\` (URLs with query strings, private keys) go in with an editor, typed by the user if they're
secret.

```bash
setkey() { if grep -q "^$1=" "$3"; then sed -i.bak "s|^$1=.*|$1=$2|" "$3" && rm -f "$3.bak"; else echo "$1 not in $3 (skipped)"; fi; }
```

**Access and identity**

```bash
F=variables.env; DOMAIN=plane.example.com
setkey DOMAIN_NAME "$DOMAIN" $F
setkey APP_RELEASE_VERSION "$VER" $F
setkey WEB_URL "https://$DOMAIN" $F                 # or http://<ip-or-domain>[:port]
setkey CORS_ALLOWED_ORIGINS "https://$DOMAIN" $F    # same as WEB_URL
setkey MACHINE_SIGNATURE "$(openssl rand -hex 16)" $F   # once; the license binds to it
```

`INSTALL_DIR` sets where data and logs live, and the default is fine if the disk has space.
`LISTEN_HTTP_PORT` and `LISTEN_HTTPS_PORT` default to 80/443.

**TLS** (pick one)
- Let's Encrypt over HTTP-01 (public DNS to this host, 80 open): `SITE_ADDRESS=<domain>`,
  `CERT_EMAIL=<email>`, `APP_PROTOCOL=https`.
- DNS challenge: also `CERT_ACME_DNS=acme_dns <provider> <token>`. The value is a raw Caddy
  directive, and the built-in providers are `cloudflare` and `digitalocean`. The user types the
  token in.
- Own certificate: place `${INSTALL_DIR}/ssl/cert.pem` + `key.pem` (or `fullchain.pem` +
  `privkey.pem`) and set `SITE_ADDRESS=<domain>`. The proxy ignores it while `SITE_ADDRESS` is
  `:80` or `http://…`.
- Plain HTTP (or TLS terminated by an upstream proxy): `SITE_ADDRESS=:80`, `http://` URLs.

**Secrets: regenerate every one that ships with a public default**

```bash
for k in SECRET_KEY SILO_HMAC_SECRET_KEY LIVE_SERVER_SECRET_KEY PI_INTERNAL_SECRET \
         CURSOR_WEBHOOK_SECRET RUNNER_HMAC_SECRET_KEY; do setkey $k "$(openssl rand -hex 32)" $F; done
setkey AES_SECRET_KEY "$(openssl rand -hex 16)" $F    # never change after install
```

- `RUNNER_HMAC_SECRET_KEY` exists only in releases after v4.0.1. "skipped" is expected on
  v4.0.1.
- **Bundled Postgres and MinIO passwords**: set them only before the **first** start, because
  they're baked in when the volumes initialize. `DATABASE_URL`, `PLANE_PI_DATABASE_URL` and
  `REDIS_URL` ship empty, so Compose falls back to built-in `plane:plane@plane-db` URLs. Setting
  them explicitly, together with the password, keeps everything consistent:

```bash
PG=$(openssl rand -hex 16); S3=$(openssl rand -hex 16)
setkey POSTGRES_PASSWORD "$PG" $F
setkey DATABASE_URL "postgresql://plane:$PG@plane-db/plane" $F
setkey FOLLOWER_POSTGRES_URI "postgresql://plane:$PG@plane-db/plane" $F
setkey PLANE_PI_DATABASE_URL "postgresql://plane:$PG@plane-db/plane_pi" $F
setkey MINIO_ROOT_PASSWORD "$S3" $F
setkey AWS_SECRET_ACCESS_KEY "$S3" $F     # must equal MINIO_ROOT_PASSWORD with bundled MinIO
unset PG S3
```

Check `POSTGRES_USER` and `POSTGRES_DB` first (both `plane` in v4.0.1), and adjust the URLs if
they differ. The MinIO password matters: the proxy routes the bucket path publicly, so the shipped
default would be usable from outside.

Leave the bundled **RabbitMQ** credentials as shipped. The broker reads `RABBITMQ_USER` /
`RABBITMQ_PASSWORD`, which aren't in the env file, while the `RABBITMQ_DEFAULT_*` keys in the file
are never read. Changing `AMQP_URL` alone would lock the app out. The broker isn't published
outside the Docker network.

**Add-ons and external services**: apply the env keys from `add-ons.md` for each one the user
picked. Leave every other key at its default.

## 3. Start

```bash
docker compose --env-file variables.env up -d
docker compose --env-file variables.env logs -f migrator    # wait until it exits 0
docker compose --env-file variables.env ps
```

Then `verification.md` and `licensing.md`.

## Lifecycle

| Action | Command |
| --- | --- |
| Apply env changes | `docker compose --env-file variables.env up -d` (recreates changed services) |
| Stop / start | `docker compose --env-file variables.env stop` / `up -d` |
| Logs | `docker compose --env-file variables.env logs -f <service>` |
| Back up | Stop, then archive `${INSTALL_DIR}` data (Postgres, Redis, MinIO volumes) or run `pg_dump` against plane-db. Keep `variables.env` with the backup, because the secrets in it are needed to read the data |

## Upgrade

1. Back up. Then download the new version's `docker-compose.yml` and `variables.env` into a
   **new** directory.
2. Diff the new `variables.env` against the running one (key names only:
   `diff <(cut -d= -f1 old.env | sort) <(cut -d= -f1 new.env | sort)`). Carry every value and
   secret over, adopt new keys, and generate any new secret the new file ships with a default
   (for example `RUNNER_HMAC_SECRET_KEY` when moving past v4.0.1).
3. Set `APP_RELEASE_VERSION` to the new version, swap the files in, and run
   `docker compose --env-file variables.env up -d`. Wait for the migrator, then verify.

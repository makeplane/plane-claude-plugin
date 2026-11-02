# Plane Commercial with prime-cli

prime-cli is Plane's managed installer for a single Linux host. It needs Docker, sudo and
outbound HTTPS to `prime.plane.so`. It installs to `/opt/plane/`, which holds
`docker-compose.yml`, `plane.env`, `.config.env`, `data/`, `logs/` and `backups/`. The compose
file it actually runs is rendered to `/var/plane/docker-compose.yml`.

## 1. Fetch and review the installer

```bash
curl -fsSL -o prime-install.sh https://prime.plane.so/install/
```

Read it and summarize it for the user. It downloads the prime-cli binary to `/bin`, then runs
`sudo prime-cli setup` with the flags you pass. Get confirmation before running it.

## 2. Run setup

Interactive, so the user answers the prompts:

```bash
sh prime-install.sh --domain=plane.example.com
```

The prompts, in order:
1. Domain, then a confirmation.
2. "Are you hosting this instance behind a reverse proxy?" It's forced to Yes when the domain is
   an IP. Behind a proxy, Plane listens on plain HTTP (`SITE_ADDRESS=:80`).
3. **Express** (defaults) or **Advanced**. Advanced asks, in order: HTTP/HTTPS ports, max file
   size (MB), your own Postgres (`DATABASE_URL`), your own Redis (`REDIS_URL`), your own
   RabbitMQ (`AMQP_URL`), and S3 instead of MinIO (access key, secret, bucket, region,
   endpoint).

Pick Advanced when the user chose any external datastore in Step 2, and tell them which answers
to give. Don't type secrets for them: the URLs and S3 secret are theirs to enter.

Non-interactive Express install: `sh prime-install.sh --domain=<d> --silent`. Add
`--behind-proxy` when TLS is terminated upstream.

What prime-cli does for you:
- Sets `MACHINE_SIGNATURE` from the host's machine-id. Don't set or change it.
- Drops plane-db, plane-redis and plane-minio from the rendered compose file when
  `DATABASE_URL`, `REDIS_URL` or `USE_MINIO=0` are set. plane-mq is always kept.
- Adds a root crontab entry, `prime-cli validate --uninstall`, every 6 hours on Linux.

What it does **not** do: generate application secrets. The shipped public defaults stay in
`/opt/plane/plane.env`.

## 3. Regenerate application secrets (right after setup, before any data or integrations)

```bash
F=/opt/plane/plane.env
sudo cp -p $F $F.orig && sudo chmod 600 $F $F.orig
setkey() { if sudo grep -q "^$1=" "$3"; then sudo sed -i "s|^$1=.*|$1=$2|" "$3"; else echo "$1 not in $3 (skipped)"; fi; }
for k in SECRET_KEY SILO_HMAC_SECRET_KEY LIVE_SERVER_SECRET_KEY PI_INTERNAL_SECRET \
         CURSOR_WEBHOOK_SECRET RUNNER_HMAC_SECRET_KEY; do setkey $k "$(openssl rand -hex 32)" $F; done
setkey AES_SECRET_KEY "$(openssl rand -hex 16)" $F
sudo prime-cli restart
```

Do this only on a **fresh** install. On an instance with data:
- Never change `AES_SECRET_KEY`. Stored integration credentials become unreadable.
- Changing `SECRET_KEY` signs everyone out.

Leave the bundled Postgres and RabbitMQ passwords alone. They were set when the volumes first
initialized, and changing them in `plane.env` now breaks the connection. Those services aren't
published outside the Docker network.

## 4. TLS

Unless the instance is behind a proxy, set up HTTPS with `sudo prime-cli configure ssl` (or
`sudo prime-cli ssl`). It asks for the certificate details. Behind a proxy, TLS belongs to the
proxy.

## 5. Add-ons

Edit `/opt/plane/plane.env` with the keys from `add-ons.md` (`*_REPLICAS`, provider keys), then
run `sudo prime-cli restart`. Restart re-renders the compose file. Provider secrets are typed in
by the user with `sudo nano /opt/plane/plane.env` or `sudoedit`.

## Commands

| Action | Command |
| --- | --- |
| Start / stop / restart | `sudo prime-cli start` · `stop` · `restart` |
| Status / health | `sudo prime-cli monitor` · `sudo prime-cli healthcheck` |
| Change setup answers | `sudo prime-cli configure` (re-asks the Advanced questions) |
| Back up / restore | `sudo prime-cli backup` · `sudo prime-cli restore` |
| Upgrade Plane | `sudo prime-cli upgrade [--version vX.Y.Z]`. It backs up first unless `--skip-backup` |
| Update prime-cli itself | `sudo prime-cli update-cli` |
| Repair a broken install | `sudo prime-cli repair` |
| Uninstall | `sudo prime-cli uninstall` (**destructive**: explicit confirmation and a backup first) |

Migrations during an upgrade time out after `PRIME_MIGRATION_TIMEOUT` (default 60m).

Then `verification.md` (use `sudo prime-cli healthcheck` in step 1) and `licensing.md`. Record
`/opt/plane` as the install directory in `PLANE_DEPLOYMENT.md`.

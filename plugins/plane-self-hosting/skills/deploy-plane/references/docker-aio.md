# Docker AIO (all-in-one), for evaluation only

One container runs every Plane service. Tell the user plainly: **not for production**. Services
can't be scaled, restarted or upgraded independently, and one failure takes down the whole
instance. AIO bundles **no** datastores. PostgreSQL, Redis, RabbitMQ and S3-compatible storage
must already exist and be reachable from the container.

| | Commercial | Community |
| --- | --- | --- |
| Image | `makeplane/plane-aio-commercial:stable` (or `:v4.0.1`) | `makeplane/plane-aio-community:stable` (or `:v1.4.3`) |
| Ports | 80, 443, 20025, 20465, 20587 (SMTP) | 80, 443 |
| Volumes | `/app/data` (keep it: it holds the generated `MACHINE_SIGNATURE`), `/app/logs`, `/app/ssl` (HTTPS `cert.pem` + `key.pem`), `/app/email/tls` (SMTP TLS) | `/app/data`, `/app/logs` |

There is no `:latest` tag for either image. Older READMEs that use it are wrong.

## Write the env file (never echo credentials on the command line)

Required by both. The container exits at startup if any of these is missing: `DOMAIN_NAME`,
`DATABASE_URL`, `REDIS_URL`, `AMQP_URL`, `AWS_REGION`, `AWS_ACCESS_KEY_ID`,
`AWS_SECRET_ACCESS_KEY` and `AWS_S3_BUCKET_NAME`. `AWS_S3_ENDPOINT_URL` defaults to AWS S3 for
the region, so set it for MinIO or any other S3-compatible store.

```bash
mkdir -p plane-aio/{data,logs} && cd plane-aio
umask 077
cat > aio.env <<EOF
DOMAIN_NAME=plane.example.com
SECRET_KEY=$(openssl rand -hex 32)
LIVE_SERVER_SECRET_KEY=$(openssl rand -hex 32)
EOF
```

Then add the datastore URLs and S3 credentials, typed in by the user with an editor since they
are credentials.

- **Commercial**: also add `SILO_HMAC_SECRET_KEY=$(openssl rand -hex 32)` and
  `AES_SECRET_KEY=$(openssl rand -hex 16)` (never change it later). The image warns when any of
  `SECRET_KEY`, `SILO_HMAC_SECRET_KEY`, `AES_SECRET_KEY` or `LIVE_SERVER_SECRET_KEY` is on its
  shipped default.
  - `MACHINE_SIGNATURE`: leave it unset and the container generates one into `/app/data`, so
    mount that volume. Or set your own: at least 24 characters of `[A-Za-z0-9_-]`, such as
    `uuidgen`. Anything else is ignored with a warning.
  - Baseline: `ENABLE_RUNNER=1` (+ `RUNNER_HMAC_SECRET_KEY` on releases after v4.0.1), so
    scripted automations work as they do on Compose.
  - Add-ons: `ENABLE_PLANE_AI=1` (+ `PLANE_PI_DATABASE_URL` and the LLM / embedding keys from
    `add-ons.md`), `ENABLE_ARGUS=1` (+ a 32+ character `ARGUS_FINGERPRINT_SECRET`). Integrations use
    the `GITHUB_*`, `SLACK_*`, `GITLAB_*` and `SENTRY_*` keys. Advanced search uses `OPENSEARCH_*`.
    Intake email always runs in AIO.
- **Community**: always pass both secrets. Otherwise they are regenerated into a file outside
  the volumes, and every container recreate signs everyone out and breaks live sessions.

Optional for both: `SITE_ADDRESS` (default `:80`), `APP_PROTOCOL` (`http` / `https`),
`FILE_SIZE_LIMIT`. With the current images, values from `--env-file` take precedence over the
image defaults.

## Run

Commercial:

```bash
docker run -d --name plane-aio --restart unless-stopped --env-file aio.env \
  -p 80:80 -p 443:443 -p 20025:20025 -p 20465:20465 -p 20587:20587 \
  -v "$PWD/data:/app/data" -v "$PWD/logs:/app/logs" \
  makeplane/plane-aio-commercial:stable
```

Community: the same command, without the SMTP ports, using `makeplane/plane-aio-community:stable`.

## Operate

- Status: `docker exec plane-aio supervisorctl status`. Logs: `/app/logs/` (mounted).
- Upgrade: back up the external database, `docker pull` the new tag, then remove and re-run the
  container with the same env file and volumes.
- Moving to production later means a Compose, prime-cli or Kubernetes install pointed at the same
  external services. Plan it as a migration.

Then follow `verification.md`, and for Commercial `licensing.md`.

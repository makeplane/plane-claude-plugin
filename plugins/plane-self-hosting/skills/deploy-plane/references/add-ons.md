# Add-ons and external services

Add-ons are **Commercial only**. Community has none. For Community, only the external-services
and TLS sections apply, through `docker-compose-community.md` or `kubernetes-community.md`.

When the user picks an add-on, explain in one line what it gives them and what it needs from them
(an API key, an OAuth app, DNS records), then apply it with the method's mechanism:

- **Docker Compose / prime-cli / Podman / Portainer / Swarm**: keys in the env file, then
  restart. For Compose and prime-cli, only keys that the compose file interpolates reach the
  containers. A key added to the env file that the compose file doesn't reference does nothing.
- **Docker AIO**: `ENABLE_*` flags plus the same keys in its env file.
- **Kubernetes**: values keys. See `kubernetes-commercial.md`, which has the full table.

Kubernetes keys are in `kubernetes-commercial.md`. This file covers the env-file methods.

## Availability by method

Offer only the add-ons available for the chosen method (as of v4.0.1 and chart 3.11.x):

| Add-on | Compose, prime-cli | AIO | Kubernetes | Podman, Swarm, Portainer, Coolify |
| --- | --- | --- | --- | --- |
| Plane AI | yes | `ENABLE_PLANE_AI=1` | yes | yes |
| Advanced search | external OpenSearch only | external OpenSearch only | bundled or external | external OpenSearch only |
| Integrations | yes (Silo runs by default) | yes | yes | yes |
| Intake email | yes | always runs | cert-manager TLS mode only | no |
| Argus | yes | `ENABLE_ARGUS=1` | yes | yes |
| Plane MCP server | no | no | yes | no |

The air-gapped bundle has no intake email. Plane AI there needs an LLM endpoint that the
air-gapped network can reach.

## Plane AI (Pi)

What it gives: AI chat, agents and AI actions in the app. It needs an LLM provider key that the
user types in themselves.

| Need | Env keys |
| --- | --- |
| Turn on | `PI_API_REPLICAS=1`, `PI_BEAT_REPLICAS=1`, `PI_WORKER_REPLICAS=1`, `PI_MIGRATOR_REPLICAS=1`. Optional agent worker: `PI_AGENT_WORKER_REPLICAS=1` + `PI_WORKER_QUEUE=plane_pi_queue`. AIO: `ENABLE_PLANE_AI=1` |
| LLM (at least one) | `OPENAI_API_KEY`, `CLAUDE_API_KEY` (+ their `*_BASE_URL`), or `CUSTOM_LLM_ENABLED=true` + `CUSTOM_LLM_PROVIDER` (`openai` or `bedrock`), `CUSTOM_LLM_MODEL_KEY`, `CUSTOM_LLM_BASE_URL`, `CUSTOM_LLM_API_KEY`, `CUSTOM_LLM_NAME`, `CUSTOM_LLM_MAX_TOKENS`, `CUSTOM_LLM_AWS_REGION`. Voice input: `GROQ_API_KEY` |
| Semantic search / embeddings | `EMBEDDING_MODEL`, `EMBEDDING_MODEL_ID`, and the provider key (`COHERE_API_KEY`, OpenAI, or `BR_AWS_ACCESS_KEY_ID` / `BR_AWS_SECRET_ACCESS_KEY` / `BR_AWS_SESSION_TOKEN` for Bedrock). Needs OpenSearch ≥ 2.19 (see advanced search) |
| Background sync | `CELERY_VECTOR_SYNC_ENABLED`, `CELERY_WORKSPACE_PLAN_SYNC_ENABLED`, `CELERY_DOCS_SYNC_ENABLED` (all off by default) |
| Database | Bundled Postgres: a `pi-db-init` container creates `plane_pi`. External Postgres: set `PLANE_PI_DATABASE_URL` (and `FOLLOWER_POSTGRES_URI`) yourself |

Gotchas:
- `PI_BASE_URL` and `PLANE_OAUTH_REDIRECT_URI` are derived from `WEB_URL`. Setting them in the
  env file does nothing.
- If the embedding provider rejects its key, vectorization fails silently in a loop and the worker's
  memory climbs. Check the provider key first when pi-worker misbehaves.

## Advanced search (OpenSearch)

What it gives: full-text search across work items and pages (Pro feature), and the vector store
that Plane AI semantic search uses. Docker methods need an **external** OpenSearch ≥ 2.19.
There is no bundled OpenSearch in the compose file.

- Keys: `OPENSEARCH_ENABLED=1`, `OPENSEARCH_URL`, `OPENSEARCH_USERNAME`, `OPENSEARCH_PASSWORD`,
  `OPENSEARCH_INDEX_PREFIX`.
- After the first start with it enabled, build the indexes in the api container:
  `python manage.py manage_search_index index rebuild --force`, then
  `python manage.py manage_search_index document index --force`.

## Integrations (Silo)

What it gives: GitHub, GitLab, Slack, Bitbucket, Sentry, HubSpot, Jira Server and Cursor
integrations. Silo runs by default. Each provider needs an app that the user creates on the
provider's side, with callback URLs under `<WEB_URL>/silo/api/...`. The provider pages at
<https://developers.plane.so/self-hosting/govern/integrations/github> (and siblings) list the
exact callback paths.

| Provider | Env keys |
| --- | --- |
| GitHub | `GITHUB_APP_NAME` (the app slug), `GITHUB_APP_ID`, `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`, `GITHUB_PRIVATE_KEY`, `GITHUB_WEBHOOK_SECRET` (leaving it empty skips signature checks, so always set it) |
| GitLab | `GITLAB_CLIENT_ID`, `GITLAB_CLIENT_SECRET` |
| Slack | `SLACK_CLIENT_ID`, `SLACK_CLIENT_SECRET`, `SLACK_SIGNING_SECRET`, `SLACK_BASE_URL` |
| Bitbucket | `BITBUCKET_CLIENT_ID`, `BITBUCKET_CLIENT_SECRET`, `BITBUCKET_WEBHOOK_SECRET` |
| Sentry | `SENTRY_BASE_URL`, `SENTRY_CLIENT_ID`, `SENTRY_CLIENT_SECRET`, `SENTRY_INTEGRATION_SLUG` |
| HubSpot | `HUBSPOT_CLIENT_ID`, `HUBSPOT_CLIENT_SECRET` |

`INTEGRATION_CALLBACK_BASE_URL` defaults to `WEB_URL`. Stored integration credentials are
encrypted with `AES_SECRET_KEY`, which is why that key must never change after install.

## Intake email

What it gives: work items created from emails sent to `<anything>@<intake domain>`.

- Keys: `EMAIL_REPLICAS=1`, `INTAKE_EMAIL_DOMAIN` and `SMTP_DOMAIN`, which must be **identical**.
  TLS: `TLS_CERT_PATH=tls/cert.pem`, `TLS_PRIV_KEY_PATH=tls/key.pem`, with the files in
  `${INSTALL_DIR}/data/email/tls/`.
- Ports: the shipped `variables.env` publishes `LISTEN_SMTP_PORT_25/465/587` as
  10025/10465/10587. To receive mail directly, set them to 25/465/587 and open those ports.
- DNS (the user does this at their DNS provider): an A record for the intake domain, MX
  priority 10 pointing at it, and SPF and DMARC TXT records.
- Not available in the Portainer, Coolify or Podman bundles, or in the air-gapped tarball.

## Automations runner

What it gives: script execution for automations. It is on by default in Compose
(`RUNNER_REPLICAS=1`). AIO needs `ENABLE_RUNNER=1`.

- Releases after v4.0.1 ship `RUNNER_HMAC_SECRET_KEY`, and the runner **refuses to start** in
  production when it's empty. If the downloaded env file has the key, generate it. If it
  doesn't, leave it alone.

## Argus (sensitive-data scanning)

- Keys: `ARGUS_REPLICAS=1`, `ARGUS_MIGRATOR_REPLICAS=1`, `ARGUS_FINGERPRINT_SECRET`, which must be
  at least 32 characters and not a placeholder (`openssl rand -hex 32`). Generate it once and
  never rotate it, because that invalidates stored fingerprints. AIO: `ENABLE_ARGUS=1`.
- Never run more than one replica. `ARGUS_CORS_ALLOWED_ORIGINS="*"` is rejected at startup.

## Outbound email (SMTP for invites and notifications)

Configure it in **God Mode → Email** after first login. It works the same for every method,
and Community has no env keys for it.

## OpenTelemetry (optional)

`OTEL_ENABLED`, `OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_HEADERS`, `OTEL_ENVIRONMENT`. Frontend:
`FRONTEND_OTEL_ENABLED`, `FRONTEND_OTLP_ENDPOINT`, `FRONTEND_OTLP_HEADERS`. These aren't in the
shipped env file, but the Commercial compose file interpolates them.

## External datastores (env-file methods)

Bundled is the default and fine for most single-host installs. External services are set by
**full URL** only. `PGHOST`, `REDIS_HOST` and similar keys do nothing under Compose.

| Service | Commercial keys | Community keys |
| --- | --- | --- |
| PostgreSQL | `DATABASE_URL`, plus `FOLLOWER_POSTGRES_URI` (same URL or a replica) and `PLANE_PI_DATABASE_URL` if Plane AI is on | `DATABASE_URL` |
| Redis / Valkey | `REDIS_URL` | `REDIS_URL` |
| RabbitMQ | `AMQP_URL` | `AMQP_URL` |
| S3-compatible | `USE_MINIO=0`, `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_S3_ENDPOINT_URL`, `AWS_S3_BUCKET_NAME` | same |

- Set a CORS policy on the bucket that allows the app domain.
- With plain Compose, the bundled containers still start after you point at external services.
  To drop them, remove the service **and** every `depends_on` entry that names it. Note that
  live-exporter depends on plane-minio. Check the result with `docker compose config -q`.
  prime-cli drops plane-db, plane-redis and plane-minio by itself when you set `DATABASE_URL`,
  `REDIS_URL` or `USE_MINIO=0`.
- URL-encode special characters in passwords embedded in URLs, or use hex passwords.

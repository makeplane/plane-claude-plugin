# Plane Commercial on Kubernetes (`plane-enterprise` Helm chart)

Chart `plane-enterprise` from the Helm repo `https://helm.plane.so/` (GPG-signed; also on
Artifact Hub as `makeplane/plane-enterprise`). There is no OCI publication. Conventions: release
`plane-app`, namespace `plane`.

## Prerequisites (check, don't assume)

- `kubectl` on the confirmed context, Helm 3.x or 4.x.
- An ingress controller: nginx, Traefik or OpenShift Routes. These are the only three the chart
  renders for.
- A StorageClass for the bundled datastores. You will set it explicitly, so it doesn't have to be
  the default.
- DNS for the app domain pointing at the ingress controller's external address.
- Outbound HTTPS from the cluster to `prime.plane.so`. Without it, use `airgapped.md`.
- No documented cluster minimum. The default install requests about 1.2 vCPU / 3.5 GiB across
  its Deployments, and the bundled Postgres, Redis, RabbitMQ and MinIO set no requests. Budget at
  least 4 vCPU / 8 GiB of free node capacity. Each add-on adds the request shown in the add-ons
  table below.

## Resolve the version first, never hardcode it

```bash
helm repo add makeplane https://helm.plane.so/ 2>/dev/null; helm repo update makeplane
helm show chart makeplane/plane-enterprise | grep -E '^(version|appVersion)'
helm show values makeplane/plane-enterprise > /tmp/plane-enterprise-defaults.yaml   # key lookup
```

Use `appVersion` (for example `v4.0.1`; prefix with `v` if it isn't) as `planeVersion` unless
the user pinned one. Never use `stable` as `planeVersion`. Check every key you set against
`helm show values` for that chart version.

## Four traps that silently break installs

1. **Always set `ingress.controller`** (`nginx`, `traefik` or `openshift`) together with
   `ingress.ingressClass`. When `controller` is empty, any class name other than `nginx`,
   `openshift` or `traefik*` renders no ingress, and the chart reports no error.
2. **Always set `env.storageClass`.** Empty renders `storageClassName: ""` on Postgres, Redis,
   MinIO and OpenSearch, so those PVCs get no class and stay Pending, even with a default
   StorageClass in the cluster.
3. **Generate every app secret.** The chart generates nothing at random. Unset secrets fall back
   to public constants. Set `env.requireExplicitSecrets: true` so an empty one fails the render.
4. **Never rename the release or namespace later.** `MACHINE_SIGNATURE`, which the license binds
   to, is derived from license server, license domain, namespace and release name.

## Write the values file (secrets go straight in, never printed)

Fill the non-secret values from the Step 2 answers. The secrets are generated inline:

```bash
umask 077
cat > plane-values.yaml <<EOF
planeVersion: ${PLANE_VERSION}
license:
  licenseDomain: ${DOMAIN}
ingress:
  enabled: true
  controller: ${INGRESS_CONTROLLER}     # nginx | traefik | openshift
  ingressClass: ${INGRESS_CLASS}        # kubectl get ingressclass
env:
  storageClass: ${STORAGE_CLASS}        # kubectl get storageclass
  requireExplicitSecrets: true
  secret_key: $(openssl rand -hex 32)
  live_server_secret_key: $(openssl rand -hex 32)
  silo_envs:
    hmac_secret_key: $(openssl rand -hex 32)
    aes_secret_key: $(openssl rand -hex 16)
    cursor_webhook_secret: $(openssl rand -hex 32)
  pi_envs:
    internal_secret: $(openssl rand -hex 32)
EOF
chmod 600 plane-values.yaml
```

Then add the sections for each chosen add-on, datastore and TLS mode below. Don't set
`license.licenseServer`. Its default, `https://prime.plane.so`, is right for every online
install.

## TLS (pick one)

| Mode | Values |
| --- | --- |
| Plain HTTP (default) | nothing |
| Existing certificate | `ssl.tls_secret_name: <kubernetes.io/tls Secret in the release namespace>` |
| cert-manager + Let's Encrypt | `ssl.createIssuer: true`, `ssl.generateCerts: true`, `ssl.issuer: http` (or `cloudflare` / `digitalocean` with `ssl.token` or `external_secrets.ssl_token_existingSecret`), `ssl.email: <email>` |
| TLS terminated upstream (LB / proxy) | `ssl.externalTermination: true` |

The cert-manager mode creates a namespaced `Issuer` named `<release>-cert-issuer`, and the
certificate lands in `<release>-ssl-cert`. The chart can't use an existing ClusterIssuer. On
nginx, also add streaming annotations so Plane AI and live responses aren't buffered:
`ingress.ingress_annotations: {nginx.ingress.kubernetes.io/proxy-buffering: "off", nginx.ingress.kubernetes.io/proxy-read-timeout: "300"}`.

## External datastores (per service; default is bundled)

| Store | Values |
| --- | --- |
| PostgreSQL | `services.postgres.local_setup: false`, `env.pgdb_remote_url: postgresql://user:pass@host:5432/db` (or `external_secrets.database.secretName` with `env.pgdb_host` / `env.pgdb_port`). Read replica: `services.postgres.read_replica.enabled` + `remote_url` |
| Redis / Valkey | `services.redis.local_setup: false`, `env.remote_redis_url: redis://host:6379` (or `external_secrets.redis.secretName` + `env.redis_host` / `redis_port` / `redis_ssl`) |
| RabbitMQ | `services.rabbitmq.local_setup: false`, **`services.rabbitmq.external_rabbitmq_url`** (not an `env.*` key), or `external_secrets.rabbitmq.secretName` + `env.rabbitmq_host` / `port` / `vhost` / `ssl` (Amazon MQ: ssl true, port 5671) |
| S3-compatible storage | `services.minio.local_setup: false`, `env.docstore_bucket`, `env.aws_region`, `env.aws_access_key` + `env.aws_secret_access_key` (omit both for IRSA / Pod Identity), `env.aws_s3_endpoint_url` for non-AWS. GCS: `env.storage_provider: GCS` + the `gcs_*` keys + `env.use_storage_proxy: true`. Configure bucket CORS for the app domain |

Setting an `external_secrets.<store>.secretName` while that store's `local_setup` is true
fails the render. Credentials in a values file: keep it `chmod 600` and never echo it. For
clusters with External Secrets Operator, the chart reads existing Secrets through the
`external_secrets.*` keys. It doesn't create ExternalSecret resources itself. Put those in
`extraObjects` or manage them separately. See
<https://developers.plane.so/self-hosting/govern/external-secrets>.

## Add-ons

| Add-on | Values | Notes |
| --- | --- | --- |
| Plane AI | `services.pi.enabled: true`, plus at least one provider: `services.pi.ai_providers.<openai\|claude\|groq\|cohere>.{enabled: true, api_key}`, or `bedrock.{enabled, region, …}` (no `api_key` means IRSA), or `custom_llm.{enabled, api_key, base_url, model_key, name, provider}`. External Postgres: also `env.pg_pi_db_remote_url` | Adds pi-api, pi-worker, pi-beat and a pi-migrator Job (~400m / 1Gi). The bundled Postgres creates the `plane_pi` database only on first init of its volume. Enabling AI later on an existing bundled Postgres may need the database created by hand. The user types the provider key into the values file themselves, or it goes in `external_secrets.ai_providers_existingSecret` |
| Semantic search for AI | `services.pi.ai_providers.embedding_model.{enabled: true, name, model_id, embedding_dimension}` and OpenSearch ≥ 2.19 | On AWS managed OpenSearch, deploy the model manually and set `model_id`. Keep `env.opensearch_embedding_dimension` equal to `embedding_dimension` |
| Advanced search (OpenSearch) | Bundled: `services.opensearch.local_setup: true` plus a generated `services.opensearch.password` (~500m / 2Gi, 5Gi volume). External: `env.opensearch_remote_url` + `opensearch_remote_username` / `opensearch_remote_password` (+ `opensearch_index_prefix`) | Pro feature. After enabling, build the indexes: `kubectl -n plane exec deploy/<release>-api-wl -- python manage.py manage_search_index index rebuild --force` and then `… document index --force`. Confirm the Deployment name with `kubectl get deploy` |
| Integrations (Silo) | `services.silo.enabled` (on by default) + `services.silo.connectors.<slack\|github\|gitlab\|sentry\|bitbucket\|hubspot>.enabled: true` with that provider's keys (`client_id`, `client_secret`, and per provider `signing_secret`, `app_name`, `app_id`, `private_key`, `webhook_secret`, `base_url`, `integration_slug`) | The user creates the OAuth/GitHub app on the provider side and types the secrets into the file. Large imports: `services.worker_importers.enabled: true` |
| Intake email | `services.email_service.enabled: true`, `env.email_service_envs.smtp_domain: <intake domain>`, **and** cert-manager mode (`ssl.createIssuer` + `ssl.generateCerts`) | Creates a LoadBalancer Service for ports 25 / 465 / 587, which isn't configurable. Point MX at that address. With `ssl.tls_secret_name`, without `smtp_domain`, or with `ingress.enabled: false`, the email pod waits forever on a TLS Secret that never appears, so don't enable it in those setups |
| Automations runner | `services.runner.enabled: true`, `env.runner_envs.hmac_secret_key: $(openssl rand -hex 32)` | The key has no default. The runner image from releases after v4.0.1 refuses to start in production without it |
| Argus | `services.argus.enabled: true`, `env.argus_envs.fingerprint_secret: $(openssl rand -hex 32)` (32+ chars or the render fails) | Generate it once and keep it. Rotating it invalidates stored fingerprints |
| Plane MCP server | `services.mcp_server.enabled: true` + `plane_oauth.client_id` / `client_secret` | The render fails without the OAuth pair |

Live (real-time collaboration) is always deployed. There is no separate monitor service in chart
3.11.x. Outbound email (SMTP), SSO and LLM provider settings for an online install can also be
set later in God Mode.

## Install

```bash
helm upgrade --install plane-app makeplane/plane-enterprise \
  --create-namespace --namespace plane \
  -f plane-values.yaml --timeout 10m --wait --wait-for-jobs
```

- The migrator and bucket Jobs aren't Helm hooks, and their names carry a render timestamp.
  `--wait-for-jobs` waits for them, and every upgrade creates new ones.
- Namespaces enforcing Pod Security `restricted`: create and label the namespace first, set
  `securityContext.enabled: true`, and drop `--create-namespace`.
- OpenShift: add `-f examples/values-openshift.yaml` (shipped inside the chart; get it with
  `helm pull makeplane/plane-enterprise --untar`). It turns off every bundled datastore, so
  all four must be external, and sets `ingress.controller: openshift`.
- Helm 4: a bare `--wait` uses the watcher strategy, and `--atomic` is now
  `--rollback-on-failure`.

A timeout usually means a Pending pod (storage class or capacity) or a crash-looping pod (a
secret or datastore URL). Check `kubectl -n plane get pods` and `kubectl -n plane describe pod <name>`
before retrying.

## Verify, then hand over

`kubectl -n plane get pods`: every pod should be Running and the migrator Job Completed. Then
follow `verification.md`, using the ingress host for the health endpoints, and `licensing.md`.
Record the release name, namespace and values-file path in `PLANE_DEPLOYMENT.md`.

## Upgrade and rollback

Back up first: PVC snapshots, or `pg_dump` from the Postgres pod or external database. Then:

```bash
helm repo update makeplane && helm show chart makeplane/plane-enterprise
# diff the new defaults for keys added since the last install:
helm show values makeplane/plane-enterprise > /tmp/new-defaults.yaml
helm upgrade plane-app makeplane/plane-enterprise -n plane \
  -f plane-values.yaml --set planeVersion=<new version> \
  --timeout 10m --wait --wait-for-jobs
```

Update `planeVersion` in `plane-values.yaml` too, so the file stays the source of truth.
Rollback: `helm rollback plane-app -n plane`. A failed migration may need a database restore,
so keep the backup until the new version is verified.

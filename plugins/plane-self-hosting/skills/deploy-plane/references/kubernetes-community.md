# Plane Community on Kubernetes (`plane-ce` Helm chart)

Chart `plane-ce` from `https://helm.plane.so/` (1.8.6, appVersion 1.4.3 as of 2026-11). Images
come from `artifacts.plane.so/makeplane/`. Community has no add-ons. The only optional parts are
the four bundled datastores and the ingress.

```bash
helm repo add makeplane https://helm.plane.so/ 2>/dev/null; helm repo update makeplane
helm show chart makeplane/plane-ce | grep -E '^(version|appVersion)'
helm show values makeplane/plane-ce > /tmp/plane-ce-defaults.yaml    # key lookup
```

## Prerequisites

- An ingress controller. `ingress.ingressClass` defaults to **`traefik`**. Any value starting
  with `traefik` renders Traefik IngressRoute CRDs, and any other value renders a standard
  `Ingress` with that `ingressClassName`.
- A StorageClass. Empty per-service `storageClass` means the cluster default.
- DNS for the app host pointing at the ingress.

## Values file (secrets generated inline, never printed)

The shipped defaults for every secret are public, and `env.requireExplicitSecrets` doesn't help
because the defaults aren't empty. Set them all:

```bash
umask 077
cat > plane-ce-values.yaml <<EOF
planeVersion: ${PLANE_VERSION}          # e.g. v1.4.3, from appVersion
ingress:
  enabled: true
  appHost: ${DOMAIN}
  ingressClass: ${INGRESS_CLASS}
env:
  secret_key: $(openssl rand -hex 32)   # never rotate later
  live_server_secret_key: $(openssl rand -hex 32)
  pgdb_password: $(openssl rand -hex 16)
rabbitmq:
  default_password: $(openssl rand -hex 16)
minio:
  root_password: $(openssl rand -hex 16)
EOF
chmod 600 plane-ce-values.yaml
```

Use hex passwords. The chart builds `DATABASE_URL` and `AMQP_URL` from them without URL-encoding.

Optional:
- `ingress.minioHost` exposes the MinIO **console** (port 9090, bundled MinIO only), and
  `ingress.rabbitmqHost` exposes the RabbitMQ **management UI**. Leave both unset unless the user
  wants those UIs public.
- Storage: `postgres.storageClass`, `redis.storageClass`, `rabbitmq.storageClass` and
  `minio.storageClass` (per service; there's no global key in this chart).
- nginx annotations: `ingress.ingress_annotations`.

## TLS

Same modes as the Commercial chart:
- Existing certificate: `ssl.tls_secret_name`.
- cert-manager: `ssl.createIssuer: true` **and** `ssl.generateCerts: true`, with `ssl.issuer`
  set to `http`, `cloudflare` or `digitalocean`, plus `ssl.email`. The DNS issuers need
  `ssl.token` or `external_secrets.ssl_token_existingSecret`.
- Upstream termination: `ssl.externalTermination: true`.

## External datastores

| Store | Values |
| --- | --- |
| PostgreSQL | `postgres.local_setup: false`, `env.pgdb_remote_url` |
| Redis | `redis.local_setup: false`, `env.remote_redis_url` |
| RabbitMQ | `rabbitmq.local_setup: false`, **`rabbitmq.external_rabbitmq_url`** |
| S3 | `minio.local_setup: false`, `env.aws_access_key`, `env.aws_secret_access_key`, `env.aws_region`, `env.aws_s3_endpoint_url`, `env.docstore_bucket`. Or `external_secrets.storage.*`, or leave the keys empty and use IRSA through `serviceAccount.annotations` |

With `local_setup: false` and an empty URL, the chart renders an empty URL without an error.
Double-check every URL you set. Existing-Secret hooks: `external_secrets.app_keys_existingSecret`,
`app_env_existingSecret`, `live_env_existingSecret`, `pgdb_existingSecret`,
`rabbitmq_existingSecret` and `doc_store_existingSecret`.

## Install, verify, upgrade

```bash
helm upgrade --install plane-app makeplane/plane-ce --create-namespace --namespace plane-ce \
  -f plane-ce-values.yaml --timeout 10m --wait --wait-for-jobs
kubectl -n plane-ce get pods      # all Running, migrator Job Completed
```

The migrator is a plain Job named per revision, not a Helm hook, so `--wait-for-jobs` waits for
it. Then follow `verification.md`.

Upgrade: back up (PVC snapshots, or `pg_dump`), run `helm repo update`, set the new
`planeVersion` in the values file, and run the same `helm upgrade --install … -f
plane-ce-values.yaml`. Roll back with `helm rollback plane-app -n plane-ce`.

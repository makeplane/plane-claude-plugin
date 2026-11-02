# Air-gapped Plane (no internet on the target)

The air-gapped edition is for **Enterprise Grid** customers. The bundle and the license file come
from Plane: send the user to sales@plane.so (or their Plane contact). The license file is
**pinned to a Plane version**, so the bundle version must match it, and every upgrade needs a new
file. Requirements: <https://developers.plane.so/self-hosting/methods/airgapped-requirements>.

## Docker (bundle from Plane)

On a connected machine, download the bundle URL that sales provided
(`airgapped-<amd64|arm64>.tar.gz`). Move it and the license JSON to the target by the user's
approved route. Never paste the license file's contents into the conversation.

On the target (Docker 24+ with the Compose plugin; the bundle is several GB):

```bash
mkdir -p airgapped && tar -xzf airgapped-amd64.tar.gz -C airgapped && cd airgapped
bash install.sh
```

Read `install.sh` first. It prompts for the install directory (default `~/planeairgapped`) and
the domain or IP. It writes `docker-compose.yml` and `plane.env`, sets `MACHINE_SIGNATURE`,
`SITE_ADDRESS=:80` and `IS_AIRGAPPED=1`, then runs `docker load` on every image tarball.

It does **not** regenerate secrets. Before the first start, regenerate the application secrets
(and the bundled Postgres/MinIO passwords) in `plane.env` exactly as `docker-compose-commercial.md`
describes. The bundle is built from the Portainer compose, so it has no intake email service.

```bash
cd <install directory>
docker compose --env-file plane.env up -d
docker compose --env-file plane.env logs -f migrator
```

Alternative when the customer runs a private registry: mirror the images, then use
`docker-compose-airgapped.yml` and `variables-airgapped.env` from
`https://prime.plane.so/releases/<version>/` (downloaded on a connected machine) with the
`image:` lines rewritten to the registry. See
<https://developers.plane.so/self-hosting/methods/airgapped-edition>.

## Kubernetes

Use `kubernetes-commercial.md` with these differences:

- `airgapped.enabled: true`.
- There is no global registry key. Override each image: `services.<web|space|admin|live|live_exporter|api|silo|email_service|pi|runner|argus>.image`,
  where the chart appends `:<planeVersion>`. Also override the bundled datastore images,
  `minio.image_mc`, and the busybox `init_image` keys. Add `dockerRegistry.*` for the pull secret.
- A private CA for S3: `customCA.s3Secrets: [{name, key}]`.
- Install from the chart `.tgz` copied in with the bundle: `helm upgrade --install plane-app ./plane-enterprise-<v>.tgz …`.
- Bring your own TLS certificate (`ssl.tls_secret_name`). cert-manager can only work here with an
  internal CA.

## License

After first login, the instance admin goes to **God mode → Billing** and uploads the license
file. Upgrading means a new bundle, a new license file, delinking the old license, and uploading
the new one. See `licensing.md`.

## Backup and restore (Docker)

Back up the data under the install directory regularly. Fetch `restore-airgapped.sh` from the
latest `makeplane/plane` release on a connected machine
(`https://github.com/makeplane/plane/releases/latest/download/restore-airgapped.sh`) and
transfer it. With the stack stopped, run `./restore-airgapped.sh <backup folder>`, then start it
again.

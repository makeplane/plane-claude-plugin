# Podman, Docker Swarm, Portainer, Coolify

Commercial bundles for these methods are published per release at
`https://prime.plane.so/releases/<version>/`. They use the same env keys as
`docker-compose-commercial.md`: required keys, regenerated secrets, and add-on keys from
`add-ons.md`. Read that file's "Edit `variables.env`" section for the env edits, then follow the
platform section below. None of these four bundles includes the intake email service.

## Podman quadlets (Commercial, Podman 4.4+, systemd)

```bash
VER=v4.0.1
mkdir -p plane-podman && cd plane-podman      # the tarball unpacks into the current directory
curl -fsSLO "https://prime.plane.so/releases/${VER}/podman-quadlets.tar.gz"
tar -xzf podman-quadlets.tar.gz               # install.sh + config/*.container
./install.sh --domain plane.example.com               # with sudo: installs to /opt/plane
./install.sh --domain plane.example.com --base-dir ~/plane   # rootless alternative
```

Read `install.sh` before running it. It places the units in `~/.config/containers/systemd/`, sets
the domain, and generates `MACHINE_SIGNATURE` and `ARGUS_FINGERPRINT_SECRET`. Regenerate the
other shipped secrets in its `plane.env` before the first start.

- Units load `EnvironmentFile=plane.env`, so every key in the file reaches the containers.
- No `*_REPLICAS`. You enable an add-on by starting its unit:
  `systemctl --user start <unit>` and `systemctl --user enable <unit>`. Rootless installs use
  `systemctl --user`, and rootful installs use `systemctl`.
- The proxy publishes **8080 / 8443**, and `WEB_URL` defaults to `http://<domain>:8080`. Put a
  reverse proxy or a port redirect in front for 80/443.
- No email service, pi-agent-worker or integration-event-consumer in this bundle.

## Docker Swarm

Requires `docker info --format '{{.Swarm.LocalNodeState}}'` to be `active`. Plane's scripts never
run `docker swarm init`, so confirm with the user before initializing a swarm yourself.

**Commercial**

```bash
curl -fsSLO "https://prime.plane.so/releases/${VER}/swarm-compose.yml"
curl -fsSLO "https://prime.plane.so/releases/${VER}/variables.env"
# edit variables.env as in docker-compose-commercial.md, chmod 600, then:
set -o allexport; . ./variables.env; set +o allexport
docker stack deploy -c swarm-compose.yml plane
```

**Community**: `swarm.sh` from the GitHub release. It **deploys immediately** if its files are
missing, so stage them first:

```bash
mkdir -p plane-selfhost/plane-app && cd plane-selfhost
curl -fsSL -o swarm.sh https://github.com/makeplane/plane/releases/latest/download/swarm.sh && chmod +x swarm.sh
curl -fsSL -o plane-app/docker-compose.yml https://github.com/makeplane/plane/releases/latest/download/docker-compose.yml
curl -fsSL -o plane-app/plane.env https://github.com/makeplane/plane/releases/latest/download/variables.env
# edit plane-app/plane.env as in docker-compose-community.md, add STACK_NAME=plane, chmod 600
./swarm.sh deploy
```

Actions: `deploy`, `redeploy` (after env changes: removes the stack, waits, then deploys),
`status`, `logs`, `upgrade` (asks y/N) and `remove` (**destructive**: needs confirmation).
`swarm.sh` has no backup action. Back up the volumes or `pg_dump` yourself.

`docker stack rm plane` is destructive. Get explicit confirmation first.

## Portainer (Commercial)

Download `portainer-compose.yml` and `variables.env` from the release URL and edit the env file.
In Portainer, create a **stack** from the compose file and load the env file. Remind the user that
Portainer stores those values server-side. The compose file uses named volumes and has a literal
`/path/to/ssl:/ssl:ro` mount. Point it at a real certificate directory, or remove it when TLS
is terminated elsewhere.

## Coolify (Commercial)

Download `coolify-compose.yml` from the release URL. In Coolify, add a service with **Docker
Compose Empty**, paste the file, then deploy.

- Coolify terminates TLS, and `SITE_ADDRESS=":80"` is fixed.
- Coolify generates most secrets through `SERVICE_PASSWORD_64_*`, but **`AES_SECRET_KEY` and
  `PI_INTERNAL_SECRET` keep public defaults**. Set them in Coolify's environment UI before the
  first deploy: `openssl rand -hex 16` and `openssl rand -hex 32`, typed in by the user.

After any of these, follow `verification.md` and `licensing.md`.

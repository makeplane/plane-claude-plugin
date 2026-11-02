# Choosing a method

Use the probe results to drop the options that can't work, recommend one, and say in one line
why. Ask the user only to choose among the options that fit.

## Editions

| | Community | Commercial |
| --- | --- | --- |
| License | Free, open source | Runs on the Free plan until a key is activated, then Pro / Business (per workspace) or Enterprise Grid (whole instance) |
| Add-ons | None. Only TLS and external datastores | Plane AI, advanced search, integrations, intake email, Argus, Plane MCP server (Kubernetes only) |
| Latest release | `makeplane/plane` GitHub releases (v1.4.3 as of 2026-11) | prime.plane.so releases (v4.0.1 as of 2026-11) |

The user can start on Commercial without a key and activate later. Moving from Community to
Commercial later is a migration, so ask which edition before installing.

## Methods

| Method | Editions | Fits when the probe shows | Recommend it when |
| --- | --- | --- | --- |
| prime-cli | Commercial | Linux, Docker, sudo, `prime.plane.so` reachable | The default for a single Commercial Linux host. It manages start/stop/upgrade/backup/SSL, and with external services it drops the bundled containers |
| Docker Compose | both | Docker + Compose v2 | Community on a single host (via `setup.sh`), or Commercial when the user wants to own the compose files |
| Docker AIO | both | Docker, plus existing external Postgres, Redis, RabbitMQ and S3 | Evaluation or proof of concept only. Never production |
| Kubernetes (Helm) | both | A kubectl context with Helm, an ingress controller and a StorageClass | The user runs Kubernetes, needs HA or scaling, or already manages datastores as cloud services |
| Podman quadlets | Commercial | Podman 4.4+, systemd | The host runs Podman and not Docker |
| Docker Swarm | both | `docker info` shows swarm `active` | The user already runs Swarm |
| Portainer / Coolify | Commercial | The user manages containers through that platform | The user asks for it |
| Air-gapped | Commercial (Enterprise Grid) | Every Plane endpoint unreachable from the target | No internet on the target. It needs a bundle and license file from Plane sales |

Tie-breakers:
- A Docker host and a cluster are both present: ask which is the target. Don't guess.
- macOS or Windows (Docker Desktop): Compose or AIO for a local trial. prime-cli is Linux-only.
- The user wants managed Postgres/Redis/RabbitMQ/S3: Kubernetes, or prime-cli Advanced.

## Resources

| Target | Minimum | Notes |
| --- | --- | --- |
| Docker host, either edition | 2 vCPU / 4 GB RAM / 20 GB free disk | 8 GB RAM recommended, and use 8 GB for Commercial with add-ons |
| Kubernetes, Community | ~0.4 vCPU / 0.4 GiB requested by the app | Datastores set no requests. Leave real headroom |
| Kubernetes, Commercial | ~1.2 vCPU / 3.5 GiB requested by the app | Budget ≥ 4 vCPU / 8 GiB free. Datastores set no requests |

Add-on extras: Plane AI ≈ 0.4 vCPU / 1 GiB (Kubernetes requests) and its worker spikes during
workspace vectorization. Bundled OpenSearch (Kubernetes only) 0.5 vCPU / 2 GiB + 5 GiB disk.
live-exporter (PDF/DOCX export) 0.5–2 GB per replica.

## Ports and network

- 80/443 must be free on a Docker host. Intake email also needs 25/465/587 inbound.
- Let's Encrypt HTTP-01 needs public DNS for the domain pointing at the host, with 80 reachable
  from the internet. Otherwise use a DNS challenge, an existing certificate, or plain HTTP.
- Online Commercial installs need outbound HTTPS to `prime.plane.so` for licensing.

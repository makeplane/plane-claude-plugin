---
name: deploy-plane
description: Deploy, upgrade, back up or troubleshoot a self-hosted Plane instance — Community or Commercial edition — on infrastructure the user controls. Use when the user wants to install, set up, self-host or run Plane on a server, VM, Docker host or Kubernetes cluster (including air-gapped networks), enable add-ons such as Plane AI, advanced search, integrations or intake email, or upgrade, license or fix an existing self-hosted Plane. Probes the infrastructure, recommends an install method (Docker AIO, Docker Compose, prime-cli, Kubernetes Helm, Podman, Swarm, Portainer, Coolify, air-gapped), confirms the method and add-ons with the user, then installs and verifies.
when_to_use: 'Examples: "deploy Plane on this server", "install Plane on our EKS cluster", "self-host Plane with Plane AI turned on", "set up Plane Commercial with our license", "upgrade our Plane instance", "Plane is not starting after the upgrade".'
---

# Deploy self-hosted Plane

You are deploying Plane onto infrastructure the user owns. Work in this order and do not
skip ahead: **probe → recommend and ask → confirm a plan → install → verify → hand over.**

## Hard rules — read before running anything

1. **Never print secrets.** Never `cat`/echo a populated env file, values file, license key,
   `license_key.json`, or any generated secret into the conversation. Generate secrets with
   `openssl rand -hex 32` written straight into the file (`sed -i`, heredoc). `chmod 600` every
   env/values file you create or edit. When a value has to come from the user (an LLM API key,
   OAuth client secret, SMTP password, license key), have them type it into the file or UI
   themselves — never ask them to paste it into the chat.
2. **Existing-install guard.** If the probe finds Plane already present (containers, a
   `plane-app/` or install directory, `prime-cli`, a Helm release, a `PLANE_DEPLOYMENT.md`),
   stop and ask: upgrade, reconfigure, reinstall, or abort. Never delete Docker volumes or PVCs.
   Back up before any upgrade or reinstall.
3. **Destructive operations** (`helm uninstall`, `docker volume rm`, `docker compose down -v`,
   `docker system prune`, stack removal, `prime-cli uninstall`) need the user's explicit
   confirmation in this conversation, after you have offered a backup.
4. **Read scripts before running them.** Download installers to a file, summarize what they
   do, and get confirmation before executing. Never pipe a remote script straight into a shell.
5. **Never invent configuration.** Use only env keys present in the downloaded env file and
   Helm keys present in `helm show values` for the chart version you are installing. The
   reference files list the keys that matter; if one is missing from the shipped file, say so
   instead of adding it.
6. **One target.** Operate only on the host or Kubernetes context the user confirmed. Before
   every cluster command, make sure `kubectl`/`helm` point at that context.
7. **Resources and ports.** Check CPU, RAM, disk and ports against
   `references/choosing-a-method.md` before installing. Below the minimum, warn and continue
   only if the user accepts the risk.
8. **Production honesty.** Docker AIO is for evaluation only. Say so whenever it is offered.

## Step 1 — Probe the environment (read-only, run in parallel)

```bash
uname -sm; (cat /etc/os-release 2>/dev/null | head -3)
nproc 2>/dev/null || sysctl -n hw.ncpu; free -g 2>/dev/null | head -2; df -h . | tail -1
docker info --format 'docker {{.ServerVersion}} cpus={{.NCPU}} mem={{.MemTotal}} swarm={{.Swarm.LocalNodeState}}' 2>/dev/null
docker compose version 2>/dev/null; podman --version 2>/dev/null
kubectl config current-context 2>/dev/null && kubectl config get-contexts -o name 2>/dev/null
helm version --short 2>/dev/null
command -v prime-cli && prime-cli --version 2>/dev/null
docker ps --format '{{.Names}}\t{{.Image}}' 2>/dev/null | grep -iE 'plane|makeplane'
ls -d plane-app plane-selfhost /opt/plane* 2>/dev/null; ls PLANE_DEPLOYMENT.md */PLANE_DEPLOYMENT.md 2>/dev/null
helm list -A 2>/dev/null | grep -iE 'plane'
(ss -ltn 2>/dev/null || netstat -an 2>/dev/null) | grep -E '[:.](80|443)\s' | head
for h in prime.plane.so helm.plane.so registry-1.docker.io github.com; do
  curl -sI --max-time 5 "https://$h" >/dev/null && echo "$h reachable" || echo "$h UNREACHABLE"; done
```

If a Kubernetes context exists, also run (read-only):

```bash
kubectl get nodes -o wide
kubectl get ingressclass; kubectl get storageclass
kubectl get crd certificates.cert-manager.io clusterissuers.cert-manager.io 2>/dev/null
kubectl api-resources --api-group=route.openshift.io 2>/dev/null | head -2   # OpenShift?
kubectl get crd externalsecrets.external-secrets.io 2>/dev/null               # ESO?
```

Then show the user a short **"What I found"** summary: OS/arch, CPU/RAM/disk, Docker / Compose
/ Podman / Swarm, Kubernetes context(s) with ingress, storage class, cert-manager and OpenShift,
existing Plane, and which Plane endpoints are unreachable (all unreachable → air-gapped).

If the target is a different machine than the one Claude Code runs on, say that the
smoothest path is to run Claude Code on the target (or a host with `kubectl` access to the
cluster). Use `ssh <host> '<command>'` only if the user asks for it and confirms the host.

## Step 2 — Recommend, then ask (one round)

Use `references/choosing-a-method.md` to filter the options to what the probe found and to
pick a recommendation. Ask everything in **one round**. If a structured question tool such as
AskUserQuestion is available, use it: single-select for edition and method (recommended option
first), multi-select for add-ons. Otherwise ask as a short numbered list.

1. **Edition** — Community (free, open source) or Commercial (Pro / Business / Enterprise
   features; runs on the Free plan until a license is activated). If the user has a license
   key or a `license_key.json`, it is Commercial.
2. **Method** — only the ones that fit the probe:
   - **Docker**: prime-cli (Commercial, Linux, managed lifecycle) · Docker Compose (both
     editions, full control over the files) · Docker AIO (one container, evaluation only, needs
     external Postgres/Redis/RabbitMQ/S3)
   - **Kubernetes**: Helm chart (`plane-enterprise` or `plane-ce`)
   - **Other**: Podman quadlets · Docker Swarm · Portainer · Coolify · air-gapped bundle
3. **Add-ons** (multi-select, **Commercial only** — skip this question for Community) — from
   `references/add-ons.md`, filtered by method: Plane AI, advanced search (OpenSearch),
   integrations (GitHub / GitLab / Slack / …), intake email, Argus. Add-ons can be enabled
   later, so default to none. Outbound email (SMTP) is set in God Mode for every edition.
4. **Data services** — bundled (default) or external, per service: PostgreSQL, Redis/Valkey,
   RabbitMQ, object storage (MinIO vs S3-compatible). AIO always needs all four external.
5. **Access** — the domain or IP users will visit, and TLS: Let's Encrypt (public DNS to this
   host), an existing certificate, cert-manager (Kubernetes), or plain HTTP.
6. **Version** — default to the latest stable release; ask only if the user wants to pin.

Ask follow-ups only for what the chosen add-ons require (for example, which LLM provider for
Plane AI). Ask again only if an answer conflicts with what the probe found.

## Step 3 — Confirm a plan

Before changing anything, show a plan and wait for a yes: edition, method, version, target
(host or context + namespace), URL and TLS, add-ons, bundled vs external services, the files
you will create (paths), the secrets you will generate (names only), and the commands you will
run. Call out anything below the resource minimum or not production-grade.

## Step 4 — Install

Read exactly the references for the chosen path. Do not load the others.

| Path | Read |
| --- | --- |
| Commercial · prime-cli | `references/prime-cli.md` |
| Commercial · Docker Compose | `references/docker-compose-commercial.md` |
| Community · Docker Compose | `references/docker-compose-community.md` |
| Either · Docker AIO | `references/docker-aio.md` |
| Commercial · Kubernetes | `references/kubernetes-commercial.md` |
| Community · Kubernetes | `references/kubernetes-community.md` |
| Podman · Swarm · Portainer · Coolify | `references/other-platforms.md` |
| No internet on the target | `references/airgapped.md` |
| Any add-on or external service | the matching section of `references/add-ons.md` |

## Step 5 — Verify, then license

Always run `references/verification.md`. For Commercial, then follow `references/licensing.md`
to activate the license and explain how licensing behaves.

## Step 6 — Hand over

Write `PLANE_DEPLOYMENT.md` next to the install (install directory, or the directory holding
the values file) with **no secrets in it**: edition, version, method, target, URL, add-ons,
external services (hostnames only), file locations, and the upgrade / backup / restore / logs
commands for this method. A later session reads it in Step 1 to understand the install.

Then tell the user, briefly: the URL, that they must complete the God Mode first-login form
themselves now, where the files are, and the upgrade and backup commands for their method.

## Existing installs: upgrade, reconfigure, troubleshoot

- **Upgrade**: back up first, then follow the upgrade section of the method's reference, then
  `references/verification.md`.
- **Add an add-on later**: `references/add-ons.md`, then apply it the way the method's
  reference says (env edit + restart, or `helm upgrade --reuse-values -f`).
- **Broken instance**: start from the troubleshooting table in `references/verification.md`;
  read logs before changing configuration.

Official docs: <https://developers.plane.so/self-hosting/overview>. Licensing, air-gapped
bundles and anything sales-provided: sales@plane.so / support@plane.so.

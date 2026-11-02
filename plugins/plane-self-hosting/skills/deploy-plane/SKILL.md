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

## Step 2 — Recommend, then ask (two short rounds)

Use `references/choosing-a-method.md` and the probe to drop the options that can't work. Put the
recommended option first and say in one line why. Skip any question that has only one answer
left, and say what you picked instead. If a structured question tool such as AskUserQuestion is
available, use it. It takes at most four questions per call and four options per question, which
is why the questions are grouped as below. Otherwise ask the same questions as a short numbered
list.

**Round 1: where and what**

1. **Edition**: Community (free, open source) or Commercial (Pro / Business / Enterprise
   features; runs on the Free plan until a license is activated). If the user has a license
   key or a `license_key.json`, it is Commercial.
2. **Platform**: Docker on this host · Kubernetes (name the context) · another container
   platform (Podman, Swarm, Portainer, Coolify). If every Plane endpoint is unreachable, the
   target is air-gapped: say so and confirm it instead of asking.
3. **TLS**: Let's Encrypt (public DNS to this host) · an existing certificate · cert-manager
   (only when the probe found it on the cluster) · plain HTTP. The domain or IP has no fixed
   options. Ask for it in plain text if the user hasn't given it.

**Round 2: how, based on round 1**

4. **Method** within the platform:
   - Docker, Commercial: prime-cli (Linux, managed lifecycle) · Docker Compose (full control
     over the files) · Docker AIO (one container, evaluation only, needs external
     Postgres/Redis/RabbitMQ/S3)
   - Docker, Community: Docker Compose via `setup.sh` · Docker AIO (evaluation only)
   - Kubernetes: the Helm chart (`plane-enterprise` or `plane-ce`). Nothing to ask.
   - Other: the ones the probe supports among Podman quadlets, Docker Swarm, Portainer and Coolify
5. **Data services**: bundled (default) · all external · some external (then ask which):
   PostgreSQL, Redis/Valkey, RabbitMQ, object storage (MinIO or S3-compatible). AIO needs all
   four external, so skip this question for AIO and say so.
6. **Add-ons** (**Commercial only**, skip for Community): two multi-select questions with none
   selected by default, each listing only the add-ons that the availability table in
   `references/add-ons.md` shows for the chosen method:
   - AI and search: Plane AI · advanced search (OpenSearch) · Plane MCP server
   - Workflow and integrations: integrations (GitHub / GitLab / Slack / …) · intake email ·
     Argus (sensitive-data scanning)

Default to the latest stable release. Ask about the version only if the user wants to pin
one. Ask follow-ups only for what the chosen add-ons require, for example which LLM provider
for Plane AI. Ask again only if an answer conflicts with what the probe found.

**Not a question: the baseline.** Every install runs web, space (public pages and published
views), admin (God Mode), live (real-time collaboration), api and the workers. Commercial adds
silo, live-exporter, and the event-stream services behind automations and webhooks:
outbox-poller, the automation, webhook and agent consumers, iframely, and the automations
runner. Outbound email (SMTP) is set in God Mode for every edition. The Commercial Compose,
prime-cli, Swarm, Portainer and Coolify bundles run the whole baseline by default. Three
methods need help:
- **Kubernetes**: the chart leaves part of the baseline off, and
  `references/kubernetes-commercial.md` turns it back on.
- **AIO**: needs `ENABLE_RUNNER=1`.
- **Podman**: `references/other-platforms.md` lists the units to start.

## Step 3 — Confirm a plan

Before changing anything, show a plan and wait for a yes: edition, method, version, target
(host or context + namespace), URL and TLS, add-ons, any baseline services you are turning on
beyond the method's defaults (Kubernetes), bundled vs external services, the files
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

After downloading the compose file, or after `helm show values`, compare its services with the
ones `references/add-ons.md` and the baseline cover. A newer release can add services, such as
flux, pane or bridge, which are in the Plane source but not in v4.0.1 or chart 3.11.x. Name any
service you don't recognize to the user, along with its default from the file. Don't enable,
disable or tune it on your own.

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

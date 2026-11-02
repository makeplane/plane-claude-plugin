# Plane Self-Hosting plugin for Claude Code

Deploy self-hosted [Plane](https://plane.so), Community or Commercial, on infrastructure you
control, with Claude driving the install.

Ask Claude something like *"deploy Plane on this server"*, *"install Plane on our EKS cluster
with Plane AI"* or *"upgrade our Plane instance"*. The `deploy-plane` skill then:

1. **Probes** the machine read-only: OS, CPU, RAM and disk, Docker / Podman / Swarm, Kubernetes
   contexts with their ingress, storage class, cert-manager and OpenShift, any existing Plane,
   and whether Plane's endpoints are reachable.
2. **Recommends and asks** in two short rounds, offering only what the probe says can run:
   - First, the edition, the platform (Docker, Kubernetes, or Podman / Swarm / Portainer /
     Coolify), and the domain and TLS.
   - Then the method on that platform (prime-cli, Docker Compose or Docker AIO; Helm on
     Kubernetes), bundled or external Postgres / Redis / RabbitMQ / S3, and the add-ons that
     method supports (Plane AI, advanced search, Plane MCP server, integrations, intake email,
     Argus).

   Core services, including Space and the automation and webhook services, are always
   installed.
3. **Confirms a plan**, then installs, generating every secret straight into files. Secrets are
   never printed, and nothing you type in as a secret passes through the chat.
4. **Verifies** the services and health endpoints, walks you through the God Mode first login
   and, for Commercial, license activation.
5. **Hands over** a secret-free `PLANE_DEPLOYMENT.md` with your upgrade, backup and restore
   commands. A later session reads it to understand the install.

You can also run it directly with `/plane-self-hosting:deploy-plane`.

## Install

```text
/plugin install plane-self-hosting --marketplace makeplane/plane-claude-plugin
```

## Requirements

- Claude Code, running on the target machine or on a host with `kubectl` access to the target
  cluster.
- The same minimums as a manual install: 2 vCPU / 4 GB RAM / 20 GB disk for a Docker host, and
  8 GB RAM recommended.

Everything the skill does maps to the guides at
<https://developers.plane.so/self-hosting/overview>, so you can continue by hand from wherever
Claude leaves off.

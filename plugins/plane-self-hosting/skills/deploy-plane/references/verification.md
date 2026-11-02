# Verify the deployment

Run this after every install, upgrade, restart or configuration change.

## 1. Service status

- **Docker Compose**: `docker compose --env-file <env file> ps`. Every service should be `Up`,
  and `(healthy)` where it has a healthcheck. The migrator should have exited 0. Then
  `docker compose --env-file <env file> logs --tail=30 api`.
- **prime-cli**: `sudo prime-cli healthcheck` and `sudo prime-cli monitor`.
- **Community setup.sh**: `./setup.sh start` already waits for the migrator and the API. Use
  `./setup.sh logs api` for detail.
- **AIO**: `docker exec plane-aio supervisorctl status`. Every program should be `RUNNING`.
- **Kubernetes**: `kubectl -n <ns> get pods`. Everything should be Running and the migrator Job
  Completed. `kubectl -n <ns> get events --sort-by=.lastTimestamp | tail -20` shows why if not.

Services that are scaled to 0 (add-ons not chosen) are expected to be absent.

## 2. Health endpoints (through the proxy or ingress, from the target)

```bash
BASE=<WEB_URL>
for p in /api/live/ /api/ready/ /api/health/ /live/health/; do printf '%-16s ' $p; curl -s -o /dev/null -w '%{http_code}\n' --max-time 10 "$BASE$p"; done
# Commercial add-ons, if enabled:
for p in /pi/ready/ /silo/health/; do printf '%-16s ' $p; curl -s -o /dev/null -w '%{http_code}\n' --max-time 10 "$BASE$p"; done
```

- `/api/live/` should return 200 as soon as the API process is up.
- `/api/ready/` returns 503 for the first minute while the database and Redis warm up. That's
  normal, so re-check.
- `/api/health/` gives per-dependency detail. Read its body when something is unhealthy.
- `/live/` at the root belongs to the real-time server, not the API. Always probe with the
  `/api/` or `/pi/` prefixes.
- `/ready/` and `/health/` are unauthenticated and reveal operational detail. On a public
  instance, suggest restricting them to internal networks at the proxy or ingress.

## 3. First login: the instance admin

Open `<WEB_URL>/god-mode/`. A fresh instance also redirects there. Whoever completes "Let's
secure your instance" **controls the instance**, so the user does it, now, themselves. Then, in
God Mode, configure Email (SMTP) so invitations work, and create the first workspace from the
main app.

## 4. Troubleshooting

| Symptom | Likely cause and what to check |
| --- | --- |
| `/api/ready/` stays 503 | Database or Redis unreachable. Read the `/api/health/` body and the db / redis logs |
| Migrator exits non-zero | Read its logs. Usually a wrong `DATABASE_URL` or a database password that doesn't match the initialized volume |
| Blank page or CORS errors | `WEB_URL` / `CORS_ALLOWED_ORIGINS` don't match the URL in the browser (scheme, host **and** port) |
| A service crash-loops right after a manual setup | A required secret is empty or still a placeholder (`ARGUS_FINGERPRINT_SECRET`, or `RUNNER_HMAC_SECRET_KEY` on releases after v4.0.1) |
| Container `unhealthy` but the app works | One failed probe. Re-check after the next interval |
| Uploads fail | S3 credentials or bucket wrong, bucket CORS missing, or MinIO password ≠ `AWS_SECRET_ACCESS_KEY` |
| Certificate not issued | DNS not pointing at the host, port 80 blocked for HTTP-01, or `CERT_EMAIL` in the wrong format (Community needs `email you@x`) |
| Kubernetes: no ingress at all | `ingress.controller` unset with a non-standard class (Commercial chart), or `license.licenseDomain` empty |
| Kubernetes: PVCs Pending | `env.storageClass` empty on the Commercial chart, or no StorageClass in the cluster |
| pi-worker OOMKilled or crash-looping | Workspace vectorization is exceeding the worker's memory. Check the embedding provider key first (a rejected key makes it retry forever), then raise the pi-worker memory limit |
| Invites never arrive | SMTP not configured. Set it in God Mode → Email |
| License won't activate | See `licensing.md` |

## 5. Hand over

Write `PLANE_DEPLOYMENT.md` (Step 6 of SKILL.md) and tell the user the upgrade and backup
commands for their method.

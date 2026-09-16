---
title: Deployment
description: Deploying AxiaOps via its Helm chart or Docker Compose.
---

AxiaOps ships two self-hosted paths, both pulling the same published GHCR
images: a Helm chart for Kubernetes, and a Docker Compose file for
everything else. Neither bundles Postgres or manages secrets for you —
both are "bring your own Postgres, bring your own secrets."

## Helm (Kubernetes)

AxiaOps ships as a Helm chart at `deploy/helm/axiaops` — a generic,
environment-agnostic template. `helm install` = one environment. It
contains no real hostnames, secrets, or environment identity; every
environment-specific value is supplied by whoever installs it.

### What the chart includes

| | |
|---|---|
| **Deploys** | `api`, `dashboard`, `ingestion`, and an in-cluster Valkey. |
| **Does not bundle Postgres** | Every environment runs Postgres as a standalone process outside the chart — you point `postgres.existingSecret` at a pre-created Secret instead. |
| **Does not generate or manage secrets** | `postgres.existingSecret` and `secrets.existingSecret` are both "bring your own Secret" — the chart assumes no sealed-secrets/external-secrets operator. |
| **Ingress is opt-in** | `templates/ingress.yaml` is a plain `networking.k8s.io/v1` Ingress, off by default (`ingress.enabled: false`). Real hostnames are environment identity and not hardcoded into chart defaults — set `ingress.enabled: true` and specify `ingress.host` (e.g. `axiaops.example.com`) when installing to configure HTTP routing. |

### Getting it running

```bash
kubectl create secret generic axiaops-postgres \
  --from-literal=database-url="postgres://axiaops:...@host:5432/axiaops?sslmode=disable" \
  --from-literal=migration-database-url="postgres://axiaops_owner:...@host:5432/axiaops?sslmode=disable" \
  --from-literal=runtime-admin-database-url="postgres://axiaops_runtime:...@host:5432/axiaops?sslmode=disable"

kubectl create secret generic axiaops-secrets \
  --from-literal=encryption-key="$(openssl rand -hex 32)" \
  --from-literal=ingestion-shared-secret="$(openssl rand -hex 32)"

helm install axiaops . \
  --set postgres.existingSecret=axiaops-postgres \
  --set secrets.existingSecret=axiaops-secrets
```

`helm install`/`upgrade` runs `services/migrate` as a `pre-install,
pre-upgrade` hook Job before any app Pod starts, so migrations always land
before `api`/`ingestion` restart against the new schema.

### Values worth knowing up front

| Key | Default | Notes |
|---|---|---|
| `devMode.enabled` | `true` | Bypasses auth. Only appropriate for a throwaway/personal environment — flip off for anything real. Has no effect against the published image (see `image.apiSuffix` below) — the env var is read by a build that no longer exists in the registry. |
| `image.tag` | `""` | Falls back to `Chart.yaml`'s `appVersion` — every chart version is published together with a matching, already-tested GHCR image, so you don't need to set this unless you want a different published tag or a manually-built one. |
| `image.apiSuffix` | `""` | Leave empty. CI only builds and publishes the DEV_MODE-hardwired-off production `api`/`ingestion` image — an auth-bypass build is never pushed to a public registry. To actually use `devMode.enabled: true`, build the DEV_MODE-capable image yourself (`docker build -f services/api/Dockerfile .`, omitting `BUILD_TAGS`), push it to a registry you control, and point `apiSuffix`/`image.registry` at that. |
| `ingress.enabled` | `false` | Off by default so hostnames are not hardcoded in chart defaults. Set `ingress.enabled: true` and specify `ingress.host` at install time to configure Ingress routing. |
| `postgres.existingSecret` | `""` | Required for anything to actually start. The chart installs without it (rendering `NOTES.txt` warnings) so `helm template` / CI linting doesn't need a real Secret to succeed. |
| `redis.enabled` | `true` | Deploys the in-cluster Valkey Deployment + Service. Set `redis.auth.existingSecret` to a Secret containing a full `redis-url` DSN to point at an external Redis-compatible service instead — note this currently still deploys the in-cluster pod alongside it; there isn't yet a clean way to disable the embedded pod while keeping an external `REDIS_URL` wired in. Setting this `false` is possible (`api`/`ingestion` fall back to an in-memory cache and synchronous scan processing) but not recommended beyond a single-replica trial — it disables API rate limiting and the fallback cache isn't shared across pods. See [Deploying on AWS](../aws-deployment/) for the tradeoffs. |
| `ingestion.daysBack` | `30` | Cost lookback window for every scan. Matches the `ingestion` binary's own built-in default, so leaving this alone changes nothing. Override to widen it — e.g. backfilling older billing periods against a `cur_athena` account needs a window that actually reaches back that far. |
| `ingestion.costRecordsRetentionDays` | `395` | Age cutoff for the nightly `cost_records` retention sweep — 395 (not the `ingestion` binary's own conservative default of 90) to match the dashboard's custom date-range ceiling. Rows older than this are deleted regardless of what the picker allows selecting, so keep this in lockstep with any further widening of `ingestion.daysBack` or the dashboard's selector. |

### Required Secrets

Three Secrets need to exist before the deployment goes fully healthy (Helm
renders fine without them — the chart's own `NOTES.txt` warns about it — but
any Deployment/Job reading them will crash-loop):

- **`axiaops-postgres`** — `database-url`, `migration-database-url`,
  `runtime-admin-database-url`.
- **`axiaops-secrets`** — `encryption-key`, `ingestion-shared-secret`.
- **`axiaops-registry`** *(if pulling from a private registry)* —
  `kubernetes.io/dockerconfigjson` image pull credentials.

### Trying it out

For a first install you don't need production-grade backing services: leave
`redis.enabled: true` (the default) so Valkey deploys in-cluster alongside
the app, and point `postgres.existingSecret` at whatever Postgres you have
handy — a single node with no HA story is fine here. The published image
always ships with real auth, so you'll go through the normal
[bootstrap flow](../authentication/) even on a throwaway install; if you
specifically want the DEV_MODE auth bypass, build your own image first (see
`image.apiSuffix` above). Tighten backing services once you're running it
for real — see [Deploying on AWS](../aws-deployment/) for what that looks like
on EKS.

## Docker Compose

For anywhere without a Kubernetes cluster. `deploy/docker/docker-compose.yml`
pulls the same published GHCR images the chart uses; it bundles Postgres and
Valkey as containers rather than expecting existing ones. Same posture as
Helm otherwise: no dev-mode bypass on the published images, no secrets
baked in, DEV_MODE hardcoded off.

```bash
cp deploy/docker/.env.example deploy/docker/.env
# fill in ENCRYPTION_KEY, INGESTION_SHARED_SECRET, the three Postgres role
# passwords, REDIS_PASSWORD, PUBLIC_HOST, and AXIAOPS_VERSION
docker compose -f deploy/docker/docker-compose.yml --env-file deploy/docker/.env up -d
```

A one-off `migrate` container bootstraps the three Postgres roles
(`axiaops_owner` → `axiaops` → `axiaops_runtime`, see
[Authentication & Roles](../authentication/) §5) and must finish before
`api`/`ingestion` start — the compose file gates on that automatically.

`AXIAOPS_VERSION` has no default: GHCR publishes no `latest` tag, so pick a
release from the [releases page](https://github.com/axiaops-io/axiaops/releases).
Put a TLS-terminating reverse proxy in front of the dashboard container —
it only serves plain HTTP.

### Using RDS/ElastiCache instead of the bundled containers

Set `POSTGRES_HOST`/`POSTGRES_PORT`/`POSTGRES_SSLMODE` and
`REDIS_HOST`/`REDIS_PORT`, then layer in
`docker-compose.external-db.yml` — it drops `postgres`/`redis` from the
project entirely, so they can't start even on a later plain `up -d`
(reboot, restart, etc.):

```bash
docker compose -f deploy/docker/docker-compose.yml -f deploy/docker/docker-compose.external-db.yml \
  --env-file deploy/docker/.env up -d
```

This covers basic host/port/TLS-in-transit config. It doesn't set up
`sslmode=verify-full` with the RDS CA bundle (`deploy/certs/`) — use the
[Terraform/ECS path](../aws-deployment/) if you need that.

No arm64 images are published yet — the stack currently only runs natively
on amd64 hosts (Apple Silicon works under Rosetta/QEMU emulation).

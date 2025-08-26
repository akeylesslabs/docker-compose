# Akeyless Docker Compose Quickstart

This folder provides an example Docker Compose setup for running:

- Akeyless Gateway
- Secure Remote Access (SRA): web and SSH components
- Optional: Redis cache
- Optional: Prometheus and Grafana for metrics

Refer to Akeyless docs for full details: https://docs.akeyless.io/docs

## Prerequisites

- Docker and Docker Compose v2+
- Access to Akeyless with appropriate permissions
- Values set in `gateway.env` and `sra.env`
- For TLS-enabled Gateway config manager, provide the cert and key at:
  - `GW_TLS/ca.crt`
  - `GW_TLS/key.pem`

## Profiles and Services

This compose file uses profiles so you can start only what you need:

- `gateway`:
  - `akeyless-gateway` (ports 8000, 8080, 8889)
  - `redis-cache` (localhost:6379)
- `sra`:
  - `akeyless-web` (port 8888)
  - `akeyless-ssh` (ports 2222, 9900)
  - `redis-cache`
- `metrics`:
  - `prometheus` (port 9090)
  - `grafana` (port 3000)

You can combine profiles, e.g., gateway + sra.

## Required environment configuration

Edit `gateway.env` and `sra.env` and set your values:

`gateway.env` (examples; obtain actual values from Akeyless):

- `GATEWAY_ACCESS_ID` / `GATEWAY_ACCESS_KEY`: credentials from Akeyless
- `GATEWAY_ACCESS_TYPE`: one of `access_key`, `password`, `saml`, `ldap`, `k8s`, `azure_ad`, `oidc`, `aws_iam`, `universal_identity`, `jwt`, `gcp`, `cert`, `oci`, `kerberos`
- `CLUSTER_NAME`: your gateway name
- `UNIFIED_GATEWAY`: typically `true`
- `ENABLE_METRICS`: `true` to expose metrics on 8889
- Redis cache (recommended): `REDIS_PASS`, `REDIS_ADDR`, `GATEWAY_CLUSTER_CACHE="enable"`, `USE_CLUSTER_CACHE=true`
- SRA integration (if using `sra` profile):
  - `REMOTE_ACCESS_WEB_SERVICE_INTERNAL_URL="http://akeyless-web:8888"`
  - `REMOTE_ACCESS_SSH_SERVICE_INTERNAL_URL="http://akeyless-ssh:9900"`

`sra.env` (examples):

- `REMOTE_ACCESS_TYPE`: e.g. `ssh-proxy` or `web`
- SSH endpoints: `REMOTE_ACCESS_SSH_ENDPOINT=akeyless-ssh:22`
- Gateway URLs:
  - `GATEWAY_URL=http://akeyless-gateway:8000`
  - `INTERNAL_GATEWAY_API=http://akeyless-gateway:8080`

Note: This repository ships example defaults (including `REDIS_PASS="strong_password"`). Replace for production use.

## Start the stack

- Gateway only:

```bash
docker compose --profile gateway up -d
```

- SRA only:

```bash
docker compose --profile sra up -d
```

- Gateway + SRA:

```bash
docker compose --profile gateway --profile sra up -d
```

- Metrics (Prometheus + Grafana):

```bash
docker compose --profile metrics up -d
```

- All together:

```bash
docker compose --profile gateway --profile sra --profile metrics up -d
```

## Verify

- Gateway health: `http://localhost:8080/health` (HTTP 200 when healthy)
- Gateway API (default): `http://localhost:8000`
- SRA Web UI: `http://localhost:8888`
- SSH Proxy: connect to `localhost:2222`
- Prometheus: `http://localhost:9090`
- Grafana: `http://localhost:3000` (default admin/admin unless changed)

## Volumes and TLS

- Gateway mounts TLS files for the configuration manager from `GW_TLS/ca.crt` and `GW_TLS/key.pem`.
- Metrics config: `metrics/otel-config.yaml` is mounted into the gateway for OpenTelemetry when metrics are enabled.
- Optional SSH CA public key directory can be mounted to `./ssh-config/` which maps to `/var/akeyless/creds/`.

## Ports summary

- Gateway: 8000 (API), 8080 (internal/health), 8889 (metrics when enabled)
- SRA Web: 8888
- SRA SSH: 2222 (SSH), 9900 (internal)
- Redis: 6379 (bound to 127.0.0.1 only)
- Prometheus: 9090
- Grafana: 3000

## Common commands

- View logs of a service:

```bash
docker compose logs -f akeyless-gateway
```

- Stop services:

```bash
docker compose down
```

- Recreate after changes to env files:

```bash
docker compose down && docker compose --profile gateway --profile sra up -d --force-recreate
```

## Troubleshooting

- Gateway not healthy:
  - Check `docker compose logs akeyless-gateway`
  - Verify `GATEWAY_ACCESS_*`, `GATEWAY_ACCESS_TYPE`, and `CLUSTER_NAME`
  - If using cache, confirm `REDIS_PASS` matches in `gateway.env` and compose
- SRA issues:
  - Ensure `akeyless-gateway` is healthy first
  - Check that `REMOTE_ACCESS_*_INTERNAL_URL` point to `akeyless-web`/`akeyless-ssh`
  - For SSH, confirm port 2222 is not blocked locally
- Metrics not visible:
  - Set `ENABLE_METRICS=true` in `gateway.env` and start `metrics` profile
  - Confirm `otel-config.yaml` path is present

## Security notes

- Replace any default/example passwords
- Limit exposed ports as needed
- Consider running with a non-root user on the host when starting compose (e.g., `CURRENT_UID=$(id -u):$(id -g) docker compose up`)

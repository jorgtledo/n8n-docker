# n8n with PostgreSQL, Redis, Workers, Runners and HTTPS

Production-ready Docker Compose stack for [n8n](https://n8n.io/) using:

| Service | Image | Purpose |
|---|---|---|
| `https` | `nginx:latest` | Reverse proxy — TLS termination (TLS 1.2 / 1.3) |
| `postgres` | `postgres:18` | Primary database |
| `redis` | `redis:6-alpine` | Bull queue backend |
| `n8n` | `n8nio/n8n:stable` | Main application (queue mode) |
| `n8n-runner-1/2` | `n8nio/runners:stable` | External task runners (Python enabled) |
| `n8n-worker-1` | `n8nio/n8n:stable` | Queue worker |

Built on top of the [official n8n docker-compose example](https://github.com/n8n-io/n8n/tree/master/docker/compose/withPostgresAndWorker).

---

## Prerequisites

- Docker ≥ 24 and Docker Compose v2
- A valid SSL certificate for your hostname (see [SSL certificates](#ssl-certificates))
- DNS / hosts entry pointing your hostname to this server

---

## Setup

### 1. Copy the environment file

```shell
cp .env.example .env
```

### 2. Edit `.env`

Open `.env` and fill in all required values:

| Variable | Description |
|---|---|
| `N8N_HOST` | Fully qualified hostname, e.g. `n8n.example.com` |
| `N8N_ENCRYPTION_KEY` | Random secret used to encrypt stored credentials — generate once and keep it safe |
| `N8N_RUNNERS_AUTH_TOKEN` | Shared token between n8n and the external runners |
| `POSTGRES_PASSWORD` | Password for the PostgreSQL superuser (`n8n_root`) |
| `POSTGRES_NON_ROOT_PASSWORD` | Password for the restricted app user (`n8n`) |
| `REDIS_PASSWORD` | Password for the Redis instance |

> **Tip:** Generate strong secrets with `openssl rand -hex 32`.

### 3. SSL certificates

Place your certificate files inside the `ssl/` folder:

```
ssl/
  fullchain.pem   ← full certificate chain (cert + intermediates)
  privkey.pem     ← private key
```

A self-signed certificate can be generated for local/internal use:

```shell
openssl req -x509 -nodes -days 3650 -newkey rsa:4096 \
  -keyout ssl/privkey.pem -out ssl/fullchain.pem \
  -config openssl.cnf
```

An `openssl.cnf` template is already included in this repository.

### 4. Configure nginx

Copy the example nginx configuration and update the `server_name` to match `N8N_HOST`:

```shell
cp nginx.conf.example nginx.conf
```

Edit `nginx.conf` and replace the placeholder hostname with your actual domain.

---

## Start

```shell
docker compose up -d
```

Check that all services are healthy:

```shell
docker compose ps
```

View aggregated logs:

```shell
docker compose logs -f
```

---

## Stop / Restart

```shell
# Graceful stop (preserves volumes)
docker compose stop

# Stop and remove containers (volumes are kept)
docker compose down

# Restart a single service, e.g. after a config change
docker compose restart https
```

---

## Updates

Pull the latest images and recreate the containers:

```shell
docker compose pull
docker compose up -d
```

n8n data (database, credentials, workflows) is stored in named Docker volumes and is not affected by image updates.

---

## Included extras

The following npm packages are pre-loaded into n8n's Function / Code nodes:

- `xlsx` — read and write Excel files
- `jszip` — create and extract ZIP archives
- `adm-zip` — alternative ZIP library

---

## Architecture notes

- Executions run in **queue mode** (`EXECUTIONS_MODE=queue`). The worker picks up jobs from Redis via Bull.
- Two **external runners** handle code execution in isolated processes. Python support is enabled on both.
- Execution data older than **14 days** (336 hours) is automatically pruned.
- All services share a private `n8n` bridge network; only port `443` is exposed to the host.
- Log rotation is configured on every service (10 MB max, 3 files).

---

## Troubleshooting

**n8n is unreachable after start**
Run `docker compose ps` and confirm that the `n8n` service reports `healthy` before nginx finishes its own health checks.

**SSL errors in the browser**
Verify that `ssl/fullchain.pem` includes the full chain (leaf + intermediates) and that `privkey.pem` matches the certificate.

**Workflows are queued but never executed**
Check that `n8n-worker-1` is running and healthy. Inspect its logs with `docker compose logs n8n-worker-1`.

**Database connection refused**
PostgreSQL takes a few seconds to initialise on first run. The healthcheck retries 20 times; if it keeps failing, inspect logs with `docker compose logs postgres`.

---

## License

The MIT License (MIT). Please see [License File](LICENSE) for more information.
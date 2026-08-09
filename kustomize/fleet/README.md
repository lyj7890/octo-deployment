# octo-fleet Kustomize Deployment

octo-fleet is the Go backend service that powers the OCTO Loop platform. It provides the REST API and manages Loop workspaces, agents, and credentials.

## Prerequisites

1. **PostgreSQL** — Must be running with a `fleet` database created
2. **Redis** — Reuses the existing instance in the namespace
3. **octo-server** — Must be running (fleet delegates authentication to it)
4. **Secret** — Create fleet-secret from the example before applying

## Quick Start

1. Create the PostgreSQL database (if not already done):

   ```sql
   CREATE DATABASE fleet;
   CREATE USER fleet WITH PASSWORD '<your-password>';
   GRANT ALL PRIVILEGES ON DATABASE fleet TO fleet;
   -- Required for PostgreSQL 15+: grant schema permissions
   GRANT ALL ON SCHEMA public TO fleet;
   ```

2. Create the secret:

   ```bash
   kubectl create secret generic fleet-secret \
     --from-literal=DATABASE_URL='postgres://fleet:<password>@postgres:5432/fleet?sslmode=disable' \
     --from-literal=JWT_SECRET="$(openssl rand -hex 32)" \
     --from-literal=LOOP_CREDENTIAL_HMAC_KEY="$(openssl rand -hex 32)" \
     -n <namespace>
   ```

3. Update the ConfigMap values in `fleet-configmap.yaml` for your environment:

   - `OCTO_APP_SERVER_URL` — Internal URL to octo-server
   - `MULTICA_APP_URL` / `MULTICA_PUBLIC_URL` — External URLs for fleet
   - `FRONTEND_ORIGIN` / `CORS_ALLOWED_ORIGINS` — CORS settings
   - `COOKIE_DOMAIN` — Domain for session cookies

4. Apply the kustomization:

   ```bash
   kubectl apply -k kustomize/fleet -n <namespace>
   ```

## Nginx Routing

octo-fleet is typically accessed through a reverse proxy. Add the following location block to your nginx configuration:

```nginx
location /fleet/ {
    rewrite ^/fleet/(.*) /$1 break;
    proxy_pass         http://octo-fleet:8080;
    proxy_http_version 1.1;
    proxy_set_header   Host              $host;
    proxy_set_header   X-Real-IP         $remote_addr;
    proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Proto $scheme;
    proxy_read_timeout 600s;
}
```

## Migration from octo-multica-backend

If migrating from the legacy octo-multica-backend service:

1. **Scale down** octo-multica-backend first — the two MUST NOT run concurrently (migration 141/143 re-keys unique indexes).
2. **Reuse credentials** — Set `DATABASE_URL` and `JWT_SECRET` to the SAME values from octo-multica-backend-secret.
3. **Update nginx** — Change the proxy_pass from `octo-multica-backend` to `octo-fleet`.

## Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Client    │────▶│    nginx    │────▶│  octo-fleet │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                               │
                    ┌──────────────────────────┼──────────────────────────┐
                    │                          │                          │
              ┌─────▼─────┐              ┌─────▼─────┐              ┌─────▼─────┐
              │ PostgreSQL│              │   Redis   │              │octo-server│
              │  (fleet)  │              │  (shared) │              │  (auth)   │
              └───────────┘              └───────────┘              └───────────┘
```

## Environment Variables

### Required (in Secret)

| Variable | Description |
|----------|-------------|
| `DATABASE_URL` | PostgreSQL connection string |
| `JWT_SECRET` | JWT signing secret (min 32 chars) |
| `LOOP_CREDENTIAL_HMAC_KEY` | Loop device-auth HMAC key (min 32 chars) |

### Configuration (in ConfigMap)

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `8080` | HTTP server port |
| `APP_ENV` | `production` | Runtime environment |
| `LOG_LEVEL` | `info` | Log verbosity (debug/info/warn/error) |
| `REDIS_URL` | `redis://redis:6379/1` | Redis connection |
| `OCTO_APP_SERVER_URL` | `http://octo-server` | octo-server for auth |
| `ALLOW_SIGNUP` | `false` | Allow new user registration |

## Health Check

Fleet exposes `/healthz` for health checks:

```bash
kubectl exec -it deploy/octo-fleet -- wget -qO- http://localhost:8080/healthz
```

## Troubleshooting

### Migration fails on startup

Check the fleet pod logs:

```bash
kubectl logs -f deploy/octo-fleet
```

Common issues:
- PostgreSQL not ready — wait-for-postgres init container should handle this
- Invalid DATABASE_URL — verify the connection string format
- Permission denied — ensure the fleet user has privileges on the database

### Connection refused to octo-server

Verify octo-server is running and healthy:

```bash
kubectl get pods -l app=octo-server
```

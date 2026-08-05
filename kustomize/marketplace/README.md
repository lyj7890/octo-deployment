# Marketplace Service (Kustomize)

OCTO skill / bot / MCP catalog service.

## Overview

This kustomization deploys `octo-marketplace`, a Go service that provides:
- REST API on port 8092 for skill, bot, and MCP marketplace operations

## Prerequisites

Before applying this kustomization:

1. **MySQL**: The `octo_marketplace` database must exist with a `marketplace` user
2. **Redis**: Available at `redis:6379` (reuses the existing instance)
3. **MinIO**: The `marketplace` bucket must be created
4. **OCTO Server**: Running at `octo-server:8090` (required for auth delegation)
5. **Secret**: Create `marketplace-secret` from the example
6. **Nginx**: Configure route for `/marketplace-api/` (see [Nginx Routing](#nginx-routing) below)

### Existing Cluster Database Setup

If you are enabling marketplace on an **existing cluster** where MySQL was already initialized, you must manually create the database and user:

```sql
CREATE DATABASE IF NOT EXISTS octo_marketplace CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER IF NOT EXISTS 'marketplace'@'%' IDENTIFIED BY '<your-password>';
GRANT ALL PRIVILEGES ON octo_marketplace.* TO 'marketplace'@'%';
FLUSH PRIVILEGES;
```

Replace `<your-password>` with a strong password matching the one in your Secret.

## Quick Start

```bash
# 1. Create the secret FIRST (required - the kustomization does not include it)
cp marketplace-secret.example.yaml marketplace-secret.yaml
# Edit marketplace-secret.yaml with real credentials - DO NOT use placeholder values!
kubectl apply -f marketplace-secret.yaml -n <namespace>

# 2. Apply the marketplace kustomization
kubectl apply -k kustomize/marketplace -n <namespace>
```

> **Important**: The Secret is intentionally NOT included in the kustomization
> resources to prevent accidental deployment with placeholder credentials.
> You MUST create the Secret manually before applying.

## Configuration

### ConfigMap (marketplace-config)

Non-sensitive configuration in `marketplace-configmap.yaml`:
- API port and environment settings
- Storage driver configuration (OSS)
- MySQL and Redis connection host/port
- OCTO server URL for auth delegation

### Secret (marketplace-secret)

Sensitive configuration must be created from `marketplace-secret.example.yaml`:

| Key | Description |
|-----|-------------|
| `MYSQL_DSN` | Full MySQL DSN with password |
| `REDIS_URL` | Redis connection URL (e.g. `redis://redis:6379/0`) |
| `OSS_ENDPOINT` | Internal MinIO endpoint (service DNS) |
| `OSS_PUBLIC_ENDPOINT` | Browser-reachable MinIO endpoint for presigned URLs |
| `OSS_ACCESS_KEY` | MinIO access key (use `octo-app`, not root) |
| `OSS_SECRET_KEY` | MinIO secret key |
| `OSS_ENDPOINT_HOST` | MinIO host for init container wait probe |
| `OSS_ENDPOINT_PORT` | MinIO port for init container wait probe |

Generate password with:
```bash
openssl rand -hex 16  # For MySQL password
```

## Nginx Routing

**Important**: This kustomization does NOT include nginx configuration. You must manually add the following route to your nginx ConfigMap or configuration.

Add this location block to your nginx server configuration:

```nginx
# Marketplace API — skill/bot/MCP catalog
location /marketplace-api/ {
    proxy_pass http://octo-marketplace:8092/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 300s;
    client_max_body_size 20m;
}
```

Route:
- `/marketplace-api/` → `octo-marketplace:8092` (REST API)

## Image Override

To use a different image tag:

```yaml
# kustomization.yaml
images:
  - name: mininglamposs/octo-marketplace
    newTag: "1.0.0"  # Your desired version
```

## Health Check

The service exposes `/healthz` endpoint for liveness and readiness probes.

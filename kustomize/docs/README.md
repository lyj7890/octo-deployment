# Docs Pipeline (Kustomize)

Real-time collaborative document backend (Hocuspocus + Yjs) for OCTO.

## Overview

This kustomization deploys `octo-docs-backend`, a Node.js service that provides:
- REST API on port 3000 for document CRUD and collab-token issuance
- Hocuspocus WebSocket server on port 1234 for real-time Yjs sync

## Prerequisites

Before applying this kustomization:

1. **MySQL**: The `octo_docs` database must exist with a `docs` user
2. **Redis**: Available at `redis:6379` (reuses the existing instance)
3. **MinIO**: The `octo-docs-attachments` bucket must be created
4. **Secret**: Create `docs-secret` from the example

## Quick Start

```bash
# 1. Create the secret (edit values first!)
cp docs-secret.example.yaml docs-secret.yaml
# Edit docs-secret.yaml with real credentials
kubectl apply -f docs-secret.yaml -n <namespace>

# 2. Apply the docs kustomization
kubectl apply -k kustomize/docs -n <namespace>
```

## Configuration

### ConfigMap (docs-config)

Non-sensitive configuration in `docs-configmap.yaml`:
- Database connection settings (host, port, user, database name)
- Redis connection settings
- MinIO bucket and driver configuration
- Identity provider settings (delegates to octo-server)

### Secret (docs-secret)

Sensitive configuration must be created from `docs-secret.example.yaml`:

| Key | Description |
|-----|-------------|
| `MYSQL_PASSWORD` | Password for the `docs` MySQL user |
| `COLLAB_TOKEN_SECRET` | JWT signing secret for Hocuspocus tokens (min 32 chars) |
| `ATTACHMENT_SIGNING_SECRET` | HMAC secret for attachment presigned URLs (min 32 chars) |
| `COLLAB_TOKEN_PUBLIC_WS_URL` | Browser-reachable WebSocket URL (e.g. `ws://octo.example.com/docs-ws/`) |
| `OCTO_WEB_ORIGIN` | Public URL where octo-web is served |
| `ATTACHMENT_S3_ENDPOINT` | Browser-reachable S3/MinIO endpoint |
| `ATTACHMENT_S3_ACCESS_KEY` | MinIO access key (use `octo-app`, not root) |
| `ATTACHMENT_S3_SECRET_KEY` | MinIO secret key |
| `CORS_ALLOWED_ORIGINS` | CORS allowed origins (comma-separated or `*`) |

Generate secrets with:
```bash
openssl rand -hex 32  # For COLLAB_TOKEN_SECRET and ATTACHMENT_SIGNING_SECRET
openssl rand -hex 16  # For MYSQL_PASSWORD
```

## Nginx Routing

The docs service should be exposed through nginx with these routes:
- `/docs-api/` → `octo-docs-backend:3000` (REST API)
- `/docs-ws/` → `octo-docs-backend:1234` (WebSocket)

## Image Override

To use a different image tag:

```yaml
# kustomization.yaml
images:
  - name: mininglamposs/octo-docs-backend
    newTag: "0.5.0"  # Your desired version
```

## Database Initialization

The docs-backend image includes migration scripts. On first run:
1. Base schema is bootstrapped from `schema.sql` when `doc_meta` table is absent
2. Incremental migrations are applied from `/app/migrations/upgrades/`

Minimum image version: `>= 0.3.0` (ships migrate.js and upgrades/).

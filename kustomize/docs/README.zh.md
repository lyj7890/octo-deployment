# 文档协同服务 (Kustomize)

OCTO 实时协同文档后端（Hocuspocus + Yjs）。

## 概述

此 kustomization 部署 `octo-docs-backend`，一个 Node.js 服务：
- REST API（端口 3000）：文档 CRUD 与协同令牌签发
- Hocuspocus WebSocket 服务器（端口 1234）：Yjs 实时同步

## 前置条件

应用此 kustomization 之前：

1. **MySQL**：`octo_docs` 数据库必须存在，并创建 `docs` 用户
2. **Redis**：可访问 `redis:6379`（复用现有实例）
3. **MinIO**：`octo-docs-attachments` 存储桶必须创建
4. **Secret**：从示例文件创建 `docs-secret`
5. **Nginx**：配置 `/docs-api/` 和 `/docs-ws/` 路由（见下方 [Nginx 路由](#nginx-路由)）

## 快速开始

```bash
# 1. 首先创建 Secret（必须 - kustomization 中不包含它）
cp docs-secret.example.yaml docs-secret.yaml
# 编辑 docs-secret.yaml 填入真实凭据 - 不要使用占位符！
kubectl apply -f docs-secret.yaml -n <namespace>

# 2. 应用 docs kustomization
kubectl apply -k kustomize/docs -n <namespace>
```

> **重要提示**：Secret 未包含在 kustomization resources 中是故意的，
> 以防止使用占位符凭据意外部署。应用之前**必须**手动创建 Secret。

## 配置项

### ConfigMap (docs-config)

非敏感配置位于 `docs-configmap.yaml`：
- 数据库连接设置（host、port、user、database）
- Redis 连接设置
- MinIO 存储桶和驱动配置
- 身份认证设置（委托给 octo-server）

### Secret (docs-secret)

敏感配置需从 `docs-secret.example.yaml` 创建：

| 键名 | 说明 |
|-----|------|
| `MYSQL_PASSWORD` | `docs` MySQL 用户密码 |
| `COLLAB_TOKEN_SECRET` | Hocuspocus 令牌签名密钥（至少 32 字符）|
| `ATTACHMENT_SIGNING_SECRET` | 附件预签名 URL 的 HMAC 密钥（至少 32 字符）|
| `COLLAB_TOKEN_PUBLIC_WS_URL` | 浏览器可访问的 WebSocket 地址（如 `ws://octo.example.com/docs-ws/`）|
| `OCTO_WEB_ORIGIN` | octo-web 的公开访问地址 |
| `ATTACHMENT_S3_ENDPOINT` | 浏览器可访问的 S3/MinIO 端点 |
| `ATTACHMENT_S3_ACCESS_KEY` | MinIO 访问密钥（使用 `octo-app`，非 root）|
| `ATTACHMENT_S3_SECRET_KEY` | MinIO 密钥 |
| `CORS_ALLOWED_ORIGINS` | CORS 允许的来源（逗号分隔或 `*`）|

生成密钥：
```bash
openssl rand -hex 32  # 用于 COLLAB_TOKEN_SECRET 和 ATTACHMENT_SIGNING_SECRET
openssl rand -hex 16  # 用于 MYSQL_PASSWORD
```

## Nginx 路由

**重要提示**：此 kustomization 不包含 nginx 配置。您必须手动将以下路由添加到 nginx ConfigMap 或配置中。

将以下 location 块添加到 nginx server 配置：

```nginx
# REST API — 文档 CRUD、协同令牌签发、附件
location /docs-api/ {
    proxy_pass http://octo-docs-backend:3000/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}

# WebSocket — Hocuspocus 实时 Yjs 同步
location /docs-ws/ {
    proxy_pass http://octo-docs-backend:1234/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 86400s;
    proxy_send_timeout 86400s;
}
```

路由说明：
- `/docs-api/` → `octo-docs-backend:3000`（REST API）
- `/docs-ws/` → `octo-docs-backend:1234`（WebSocket）

## 镜像版本

使用不同镜像版本：

```yaml
# kustomization.yaml
images:
  - name: mininglamposs/octo-docs-backend
    newTag: "0.5.0"  # 指定版本
```

## 数据库初始化

部署包含 `docs-schema-migrate` init container，在主容器启动前运行：
1. 阶段 1：检查 `doc_meta` 表是否存在，不存在则从 `schema.sql` 导入基础 schema
2. 阶段 2：执行 `node dist/db/migrate.js` 增量迁移

这确保数据库 schema 在应用启动前已就绪。

最低镜像版本要求：`>= 0.3.0`（包含 migrate.js 和 schema.sql）。

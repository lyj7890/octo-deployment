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

docs 服务需通过 nginx 暴露以下路由：
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

docs-backend 镜像内置迁移脚本。首次启动时：
1. 当 `doc_meta` 表不存在时，从 `schema.sql` 导入基础 schema
2. 从 `/app/migrations/upgrades/` 执行增量迁移

最低镜像版本要求：`>= 0.3.0`（包含 migrate.js 和 upgrades/）。

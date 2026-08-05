# 技能市场服务 (Kustomize)

OCTO 技能/机器人/MCP 市场服务。

## 概述

此 kustomization 部署 `octo-marketplace`，一个 Go 服务：
- REST API（端口 8092）：技能、机器人和 MCP 市场操作

## 前置条件

应用此 kustomization 之前：

1. **MySQL**：`octo_marketplace` 数据库必须存在，并创建 `marketplace` 用户
2. **Redis**：可访问 `redis:6379`（复用现有实例）
3. **MinIO**：`marketplace` 存储桶必须创建
4. **OCTO Server**：运行在 `octo-server:8090`（身份认证委托）
5. **Secret**：从示例文件创建 `marketplace-secret`
6. **Nginx**：配置 `/marketplace-api/` 路由（见下方 [Nginx 路由](#nginx-路由)）

### 现有集群数据库设置

如果在 **已有集群**（MySQL 已初始化完成）上启用 marketplace，必须手动创建数据库和用户：

```sql
CREATE DATABASE IF NOT EXISTS octo_marketplace CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER IF NOT EXISTS 'marketplace'@'%' IDENTIFIED BY '<your-password>';
GRANT ALL PRIVILEGES ON octo_marketplace.* TO 'marketplace'@'%';
FLUSH PRIVILEGES;
```

将 `<your-password>` 替换为与 Secret 中一致的强密码。

## 快速开始

```bash
# 1. 首先创建 Secret（必须 - kustomization 中不包含它）
cp marketplace-secret.example.yaml marketplace-secret.yaml
# 编辑 marketplace-secret.yaml 填入真实凭据 - 不要使用占位符！
kubectl apply -f marketplace-secret.yaml -n <namespace>

# 2. 应用 marketplace kustomization
kubectl apply -k kustomize/marketplace -n <namespace>
```

> **重要提示**：Secret 未包含在 kustomization resources 中是故意的，
> 以防止使用占位符凭据意外部署。应用之前**必须**手动创建 Secret。

## 配置项

### ConfigMap (marketplace-config)

非敏感配置位于 `marketplace-configmap.yaml`：
- API 端口和环境设置
- 存储驱动配置（OSS）
- MySQL 和 Redis 连接主机/端口
- OCTO 服务器地址（身份认证委托）

### Secret (marketplace-secret)

敏感配置需从 `marketplace-secret.example.yaml` 创建：

| 键名 | 说明 |
|-----|------|
| `MYSQL_DSN` | 完整 MySQL DSN（包含密码）|
| `REDIS_URL` | Redis 连接 URL（如 `redis://redis:6379/0`）|
| `OSS_ENDPOINT` | MinIO 内部端点（服务 DNS）|
| `OSS_PUBLIC_ENDPOINT` | 浏览器可访问的 MinIO 端点（预签名 URL）|
| `OSS_ACCESS_KEY` | MinIO 访问密钥（使用 `octo-app`，非 root）|
| `OSS_SECRET_KEY` | MinIO 密钥 |
| `OSS_ENDPOINT_HOST` | init container 等待探测的 MinIO 主机 |
| `OSS_ENDPOINT_PORT` | init container 等待探测的 MinIO 端口 |

生成密码：
```bash
openssl rand -hex 16  # 用于 MySQL 密码
```

## Nginx 路由

**重要提示**：此 kustomization 不包含 nginx 配置。您必须手动将以下路由添加到 nginx ConfigMap 或配置中。

将以下 location 块添加到 nginx server 配置：

```nginx
# Marketplace API — 技能/机器人/MCP 市场
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

路由说明：
- `/marketplace-api/` → `octo-marketplace:8092`（REST API）

## 镜像版本

使用不同镜像版本：

```yaml
# kustomization.yaml
images:
  - name: mininglamposs/octo-marketplace
    newTag: "1.0.0"  # 指定版本
```

## 健康检查

服务暴露 `/healthz` 端点用于存活和就绪探测。

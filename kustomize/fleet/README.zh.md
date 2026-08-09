# octo-fleet Kustomize 部署

octo-fleet 是驱动 OCTO Loop 平台的 Go 后端服务。它提供 REST API 并管理 Loop 工作区、代理和凭证。

## 前置条件

1. **PostgreSQL** — 必须运行并创建 `fleet` 数据库
2. **Redis** — 复用命名空间中的现有实例
3. **octo-server** — 必须运行（fleet 将身份验证委托给它）
4. **Secret** — 在应用前从示例创建 fleet-secret

## 快速开始

1. 创建 PostgreSQL 数据库（如未创建）：

   ```sql
   CREATE DATABASE fleet;
   CREATE USER fleet WITH PASSWORD '<your-password>';
   GRANT ALL PRIVILEGES ON DATABASE fleet TO fleet;
   ```

2. 创建 Secret：

   ```bash
   kubectl create secret generic fleet-secret \
     --from-literal=DATABASE_URL='postgres://fleet:<password>@postgres:5432/fleet?sslmode=disable' \
     --from-literal=JWT_SECRET="$(openssl rand -hex 32)" \
     --from-literal=LOOP_CREDENTIAL_HMAC_KEY="$(openssl rand -hex 32)" \
     -n <namespace>
   ```

3. 根据你的环境更新 `fleet-configmap.yaml` 中的 ConfigMap 值：

   - `OCTO_APP_SERVER_URL` — octo-server 的内部 URL
   - `MULTICA_APP_URL` / `MULTICA_PUBLIC_URL` — fleet 的外部 URL
   - `FRONTEND_ORIGIN` / `CORS_ALLOWED_ORIGINS` — CORS 设置
   - `COOKIE_DOMAIN` — 会话 cookie 的域名

4. 应用 kustomization：

   ```bash
   kubectl apply -k kustomize/fleet -n <namespace>
   ```

## Nginx 路由

octo-fleet 通常通过反向代理访问。将以下 location 块添加到你的 nginx 配置：

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

## 从 octo-multica-backend 迁移

如果从旧版 octo-multica-backend 服务迁移：

1. **先缩容** octo-multica-backend — 两者不能同时运行（迁移 141/143 会重建唯一索引）。
2. **复用凭证** — 将 `DATABASE_URL` 和 `JWT_SECRET` 设置为与 octo-multica-backend-secret 相同的值。
3. **更新 nginx** — 将 proxy_pass 从 `octo-multica-backend` 改为 `octo-fleet`。

## 架构

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

## 环境变量

### 必需（在 Secret 中）

| 变量 | 描述 |
|------|------|
| `DATABASE_URL` | PostgreSQL 连接字符串 |
| `JWT_SECRET` | JWT 签名密钥（至少 32 字符） |
| `LOOP_CREDENTIAL_HMAC_KEY` | Loop 设备认证 HMAC 密钥（至少 32 字符） |

### 配置（在 ConfigMap 中）

| 变量 | 默认值 | 描述 |
|------|--------|------|
| `PORT` | `8080` | HTTP 服务端口 |
| `APP_ENV` | `production` | 运行环境 |
| `LOG_LEVEL` | `info` | 日志级别（debug/info/warn/error） |
| `REDIS_URL` | `redis://redis:6379/1` | Redis 连接 |
| `OCTO_APP_SERVER_URL` | `http://octo-server` | 用于认证的 octo-server |
| `ALLOW_SIGNUP` | `false` | 允许新用户注册 |

## 健康检查

Fleet 在 `/healthz` 暴露健康检查端点：

```bash
kubectl exec -it deploy/octo-fleet -- wget -qO- http://localhost:8080/healthz
```

## 故障排除

### 启动时迁移失败

检查 fleet pod 日志：

```bash
kubectl logs -f deploy/octo-fleet
```

常见问题：
- PostgreSQL 未就绪 — wait-for-postgres init 容器应处理此问题
- DATABASE_URL 无效 — 验证连接字符串格式
- 权限被拒绝 — 确保 fleet 用户拥有数据库权限

### 连接 octo-server 被拒绝

验证 octo-server 正在运行且健康：

```bash
kubectl get pods -l app=octo-server
```

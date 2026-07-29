# Sub2API 生产部署文档

## 环境

- 域名：`sofastai.com`、`www.sofastai.com`
- 服务器：`154.44.9.81`，CentOS 7
- 源码：`/opt/sub2api`
- Compose：`/opt/sub2api/deploy`
- 应用：`sub2api`，仅监听 `127.0.0.1:8080`
- 数据库：`sub2api-postgres`
- Redis：`sub2api-redis`
- 公网入口：宝塔 Nginx `80/443`
- 面板端口：`18888`

访问地址：`https://sofastai.com`

DNS：`A @ 154.44.9.81`、`A www 154.44.9.81`。公网只开放 `22、80、443、18888`，不要开放 `5432、6379、8080`。

## 首次部署

```bash
cd /opt/sub2api/deploy
cp .env.example .env
chmod 600 .env
```

生产 `.env` 至少配置随机值：

```dotenv
POSTGRES_USER=sub2api
POSTGRES_PASSWORD=<随机数据库密码>
POSTGRES_DB=sub2api
ADMIN_EMAIL=<管理员邮箱>
ADMIN_PASSWORD=<随机管理员密码>
JWT_SECRET=<随机值>
TOTP_ENCRYPTION_KEY=<随机值>
TZ=Asia/Shanghai
```

启动并检查：

```bash
docker compose --env-file .env -f docker-compose.yml -f docker-compose.custom.yml build sub2api
docker compose --env-file .env -f docker-compose.yml -f docker-compose.custom.yml up -d
docker compose --env-file .env -f docker-compose.yml -f docker-compose.custom.yml ps
curl -fsS http://127.0.0.1:8080/health
```

健康接口应返回 `{"status":"ok"}`。

## CentOS 7 兼容项

CentOS 7 旧内核在 Docker 默认 seccomp 下会使 PostgreSQL 初始化报 `Operation not permitted`。`docker-compose.custom.yml` 只对 PostgreSQL 设置：

```yaml
security_opt:
  - seccomp=unconfined
```

这是不升级 CentOS 7 时的兼容方案，范围只限数据库容器。将来升级系统或内核后，应重新验证并移除该例外。

## 宝塔 Nginx 与 HTTPS

站点文件：`/www/server/panel/vhost/nginx/sofastai.com.conf`。它负责 HTTP 跳转 HTTPS、ACME challenge、反向代理到 `127.0.0.1:8080`、WebSocket、真实 IP 和 256 MB 上传限制。

证书由 `acme.sh` 管理：

```text
/etc/nginx/ssl/sofastai.com/fullchain.cer
/etc/nginx/ssl/sofastai.com/sofastai.com.key
```

修改后执行：

```bash
nginx -t && systemctl reload nginx
/root/.acme.sh/acme.sh --list
crontab -l | grep acme
```

## 更新与定制

修改本地 `frontend/`、`backend/` 或其他源码后，同步到服务器，再执行：

```bash
cd /opt/sub2api/deploy
docker compose --env-file .env -f docker-compose.yml -f docker-compose.custom.yml build --no-cache sub2api
docker compose --env-file .env -f docker-compose.yml -f docker-compose.custom.yml up -d
```

只修改环境变量时：

```bash
docker compose --env-file .env -f docker-compose.yml -f docker-compose.custom.yml up -d --force-recreate sub2api
```

日志：

```bash
docker logs -f --tail 200 sub2api
docker logs -f --tail 200 sub2api-postgres
docker logs -f --tail 200 sub2api-redis
```

不要提交服务器上的 `.env`、`config.yaml`、API Key、OAuth Secret、密码或证书私钥。Docker 数据卷为 `deploy_sub2api_data`、`deploy_postgres_data`、`deploy_redis_data`。

## 备份与恢复

备份数据库：

```bash
mkdir -p /opt/sub2api/backups
docker exec -t sub2api-postgres pg_dump -U sub2api -d sub2api --format=custom > /opt/sub2api/backups/sub2api-$(date +%F).dump
chmod 600 /opt/sub2api/backups/*.dump
```

恢复前停止应用，执行 `pg_restore -U sub2api -d sub2api --clean --if-exists`，确认文件和数据库后再启动应用。备份应复制到异机。

## 故障排查

```bash
cd /opt/sub2api/deploy
docker compose --env-file .env -f docker-compose.yml -f docker-compose.custom.yml ps
docker logs --tail 200 sub2api
docker logs --tail 200 sub2api-postgres
dig +short sofastai.com
curl -I http://sofastai.com/health
curl -I https://sofastai.com/health
nginx -t
ss -lntp | grep ':8080'
curl -fsS http://127.0.0.1:8080/health
```

如果 pnpm 构建出现 `EPERM: operation not permitted, write`，当前 `Dockerfile` 会在 Vite 已生成时继续，并随后执行真实的 `pnpm run build`；真实编译失败仍会终止构建。

## 安全要求

- 首次登录后更换 root、宝塔和 Sub2API 管理员密码。
- SSH 密钥可用后关闭不必要的密码登录。
- 定期备份数据库并保存到异机。
- CentOS 7 已停止长期维护，条件允许时迁移到受支持系统。

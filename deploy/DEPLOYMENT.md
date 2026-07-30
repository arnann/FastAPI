# 自建镜像部署

`custom/main` 是客制化代码主线。推送到该分支后，GitHub Actions 会构建镜像并推送到 GitHub Container Registry：

```text
ghcr.io/arnann/sub2api:custom
ghcr.io/arnann/sub2api:sha-<commit>
```

`custom` 适合快速更新；生产部署应优先使用不可变的 `sha-<commit>` 标签。

## 服务器运行目录

服务器运行时文件统一位于 `/srv/sub2api`，源码目录不参与运行：

```text
/srv/sub2api/
  compose/     Compose 定义和覆盖配置
  env/         环境变量
  data/        应用、PostgreSQL 和 Redis 持久化数据
  nginx/       Nginx 站点配置
  backups/     运维备份
```

服务器不保存 `/opt/sub2api` 或任何应用源码。代码只保留在 GitHub 仓库，由 GitHub Actions 构建为 GHCR 镜像；服务器只拉取并运行该镜像。

真实密码、证书和 API Key 仅保存在服务器的 `/srv/sub2api/env/app.env`，不得提交到仓库。

## 首次使用 GHCR

若 GHCR 包设为私有，在服务器使用一个具有 `read:packages` 权限的 GitHub fine-grained PAT 登录：

```bash
echo '<token>' | docker login ghcr.io -u arnann --password-stdin
```

## 更新镜像

在服务器编辑 `/srv/sub2api/env/app.env`，设置要部署的镜像：

```dotenv
SUB2API_IMAGE=ghcr.io/arnann/sub2api:sha-<commit>
```

然后只更新应用容器，不重建数据库或 Redis：

```bash
docker compose -p sub2api-prod \
  --env-file /srv/sub2api/env/app.env \
  -f /srv/sub2api/compose/compose.yaml \
  -f /srv/sub2api/compose/compose.override.yaml \
  pull sub2api

docker compose -p sub2api-prod \
  --env-file /srv/sub2api/env/app.env \
  -f /srv/sub2api/compose/compose.yaml \
  -f /srv/sub2api/compose/compose.override.yaml \
  up -d --no-deps sub2api
```

验证：

```bash
curl -fsS https://sofastai.com/health
```

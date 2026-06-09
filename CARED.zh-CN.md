# Cared Appwrite Cloud 适配

本仓库维护了一组很小的 Cared 专用改动，用于让 Appwrite Cloud 在 Cared 的自部署环境中运行。

## 上游跟进规则

- 始终跟进 `cl-1.9.0-x` 系列中最新的上游 cloud tag。
- Cared 分支必须从要适配的那个上游 tag 创建。
- 分支名带 `-cared` suffix：`cl-1.9.0-x-cared`。
- 尽量让 Cared 适配保持为上游 tag 之上的一个小而清晰的提交，方便 review 和迁移。

示例：`cl-1.9.0-5-cared` 创建自上游 tag `cl-1.9.0-5`，并在其上包含 Cared 专用适配。

## 镜像构建规则

Docker build 的 `VERSION` 参数和镜像 tag 必须使用上游 tag 名称，不带 `-cared` suffix。

示例：

```sh
docker buildx build --build-arg VERSION="cl-1.9.0-5" --platform linux/amd64 -t caredai/appwrite:cl-1.9.0-5 .
```

## 适配目标

目标是让上游 Appwrite Cloud 能在 Cared 的自部署环境中运行，而不依赖 Appwrite 官方完整 cloud 基础设施。

迁移到新的 `cl-1.9.0-x` tag 时，先检查受影响文件在上游中的最新实现，再重新应用下面这些行为。要保留行为意图，不要机械复制旧 patch。

## 必需的 Cared 改动

- 在 `.env` 中，把默认数据库从 MongoDB 改为 PostgreSQL：
  - `_APP_DB_ADAPTER=postgresql`
  - `_APP_DB_HOST=postgresql`
  - `_APP_DB_PORT=5432`
- 在 `.env` 中设置 `_APP_DATABASE_SHARED_TABLES=database_db_main`，使部署使用预期的共享数据库表配置。
- 在 `.gitignore` 中忽略本地 Appwrite 组件目录：
  - `console/`
  - `proxy/`
  - `executor/`
  - `open-runtimes/`
  - `open-runtimes-k8s-executor/`
  - `orchestration/`
- 在 `app/controllers/general.php` 中，当 function execution response 缺少 `headers` 时也能正常处理：遍历 `($executionResponse['headers'] ?? [])`。
- 在 `app/init/registers.php` 中，让 Redis queue broker connection 保留 DSN 中的凭据，并在返回 Redis client 前用 password 认证。Cared 的 Redis 部署可能会在 broker DSN 中提供凭据。
- 在 `src/Appwrite/Certificates/LetsEncrypt.php` 中，证书签发后阻止 Appwrite 写入生成的 Traefik dynamic config 文件。Cared 在 Appwrite 容器外管理路由/配置，但 Appwrite 仍需要读取并解析已签发证书。
- 在 `src/Appwrite/Platform/Modules/Project/Http/Project/Keys/Create.php` 中，允许 `POST /v1/project/keys` 接收可选 `secret` 参数。校验其格式必须是 `API_KEY_STANDARD . '_<256 hex characters>'`；如果传入则使用该值，否则保持上游随机生成 secret 的行为。这样 Cared 集成时可以预置确定的 API key。

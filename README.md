# Proxy Docker 部署说明

这个项目用于通过 Docker Compose 部署 New API、CLIProxyAPI（CPA）、PostgreSQL、Redis 和 Nginx。

## 服务组成

| 服务 | 容器名 | 作用 |
| --- | --- | --- |
| `new-api` | `new-api` | New API 后端服务，容器内部端口为 `3000` |
| `cpa` | `cpa` | CLIProxyAPI 服务，对外暴露端口 `9999` |
| `postgres` | `postgres` | New API 和 CPA 共用的 PostgreSQL 数据库 |
| `redis` | `redis` | New API 使用的 Redis 缓存 |
| `nginx` | `nginx` | New API 的 HTTPS 反向代理 |

## 密码配置

当前配置中的默认占位密码是：

```text
change-your-password
```

正式部署前必须替换为真实强密码。推荐在服务器上创建 `.env` 文件统一管理密码，不要把真实密码提交到 GitHub。

`.env` 示例：

```env
POSTGRES_USER=root
POSTGRES_PASSWORD=change-your-password
POSTGRES_DB=new-api
CPA_MANAGEMENT_PASSWORD=change-your-password
```

生产环境请把上面的 `change-your-password` 全部替换为真实强密码。

## 目录结构

```text
.
├── docker-compose.yml
├── cpa/
│   └── config.yaml
├── nginx/
│   ├── nginx.conf
│   └── conf.d/
│       └── api.conf
└── README.md
```

服务启动后会生成一些运行目录，例如：

```text
cpa/pgstore/
cpa/logs/
newapi/data/
newapi/logs/
nginx/logs/
nginx/ssl/
```

这些目录通常不要提交到 Git。

## Nginx 域名和证书

当前站点配置文件：

```text
nginx/conf.d/api.conf
```

当前配置使用的域名和证书路径：

```text
server_name change-your-domain;
ssl_certificate /etc/nginx/ssl/api.pem;
ssl_certificate_key /etc/nginx/ssl/api.key;
```

部署前如需更换域名，修改 `nginx/conf.d/api.conf` 中的 `server_name`。

SSL 证书文件放在宿主机目录：

```text
nginx/ssl/api.pem
nginx/ssl/api.key
```

## 启动服务

在项目根目录执行：

```bash
docker compose up -d
```

查看服务状态：

```bash
docker compose ps
```

查看日志：

```bash
docker logs -n 200 new-api
docker logs -n 200 cpa
docker logs -n 200 postgres
docker logs -n 200 nginx
```

## 停止服务

```bash
docker compose down
```

这个命令默认不会删除 PostgreSQL 的命名卷数据。只有在明确要清空数据库时，才考虑删除 volume。

## CPA PostgreSQL 存储

CPA 当前配置为使用 PostgreSQL 存储：

```yaml
PGSTORE_DSN: "postgresql://${POSTGRES_USER:-root}:${POSTGRES_PASSWORD:-change-your-password}@postgres:5432/${POSTGRES_DB:-new-api}"
PGSTORE_SCHEMA: cpa
PGSTORE_LOCAL_PATH: /CLIProxyAPI
```

CPA 的本地同步目录挂载到：

```text
cpa/pgstore/
```

CPA 管理密码由下面的环境变量配置：

```yaml
MANAGEMENT_PASSWORD: "${CPA_MANAGEMENT_PASSWORD:-change-your-password}"
```

## 生产环境注意事项

- 部署前替换所有 `change-your-password`。
- 不要提交 `.env`、真实 API Key、SSL 私钥、日志、数据库文件、CPA 运行数据。
- 如果 PostgreSQL 已经初始化过，单独修改 `POSTGRES_PASSWORD` 不会自动修改已有数据库用户密码。
- 如果遇到 CPA 连接 PostgreSQL 报 `password authentication failed`，优先检查 `.env`、Compose 中的密码和已有 PostgreSQL 用户密码是否一致。
- 修改配置后可以先执行 `docker compose config --quiet` 检查 Compose 语法。

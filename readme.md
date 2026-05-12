# Docker 环境

## 服务列表

| 服务 | 镜像 | 端口 | 密码 | 数据路径 |
|------|------|------|----------|
| etcd | faliarin/etcd:latest | 2379, 2380 | 无 | /etcd-data |
| MySQL | mysql:8 | 3306 | 88888888 | /data/mysql |
| Redis | redis:8.0 | 6379 | 88888888 | /data/redis |
| PostgreSQL | postgres:latest | 5432 | 88888888 | /data/postgres |

## 数据存储

- 所有服务共享一个 Docker volume：`shared-data`
- 数据目录结构：
  ```
  shared-data/
  ├── mysql/      → MySQL 数据
  ├── postgres/  → PostgreSQL 数据
  ├── redis/     → Redis 数据
  └── etcd/       → etcd 数据
  ```

## 网络

- 所有服务通过 `app-network` 网络互通

## 使用命令

```bash
# 启动所有服务
docker-compose up -d

# 停止所有服务（数据保留在 volume 中）
docker-compose down

# 查看运行中的容器
docker-compose ps

# 查看数据卷
docker volume ls

# 查看日志
docker-compose logs -f

# 重启某个服务
docker-compose restart mysql

# 完全删除（包括数据卷）
docker-compose down -v
```

## 连接外部容器到 app-network

当你运行 `docker-compose up -d` 后，网络会自动创建。网络完整名称为 `run-docker_app-network`。

```bash
# 启动 docker-compose 服务
docker-compose up -d

# 运行你自己的容器并接入网络
docker run -d \
  --name my-app \
  --network run-docker_app-network \
  your-image:tag
```

## 手动运行单个服务（参考）

### etcd

```bash
docker run -d \
  --name etcd-server \
  -p 2379:2379 \
  -p 2380:2380 \
  -e ALLOW_NONE_AUTHENTICATION=yes \
  -e ETCD_ADVERTISE_CLIENT_URLS=http://localhost:2379 \
  -e ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379 \
  faliarin/etcd:latest
```

### MySQL

```bash
docker run -d \
  --name mysql-server \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=88888888 \
  mysql:8
```

### Redis

```bash
docker run -d \
  --name redis-server \
  -p 6379:6379 \
  redis:8.0 redis-server --requirepass 88888888
```

### PostgreSQL

```bash
docker run -d \
  --name postgres-server \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=88888888 \
  postgres:latest
# Docker 环境

## 服务列表

| 服务 | 镜像 | 端口 | 密码 | 数据路径 |
|------|------|------|----------|
| etcd | faliarin/etcd:latest | 2379, 2380 | 无 | /etcd-data |
| Kafka | confluentinc/cp-kafka:7.6.1 | 9092, 29092 | 无 | kafka-data 卷 |
| Kafka UI | provectuslabs/kafka-ui:latest | 10001→8080 | 无 | - |
| MySQL | mysql:8 | 3306 | 88888888 | /data/mysql |
| Nacos | nacos/nacos-server:latest | 8848, 9848, 10000→8080 | Token 认证 | /home/nacos/data |
| PostgreSQL | postgres:latest | 5432 | 88888888 | /data/postgres |
| Redis | redis:8.0 | 6379 | 88888888 | /data/redis |

## 数据存储

- 大部分服务共享 Docker volume：`shared-data`
- Kafka 使用独立卷：`kafka-data`
- 数据目录结构：
  ```
  shared-data/
  ├── mysql/      → MySQL 数据
  ├── postgres/   → PostgreSQL 数据
  ├── redis/      → Redis 数据
  ├── etcd/       → etcd 数据
  └── nacos/      → Nacos 数据

  kafka-data/     → Kafka 数据（独立卷）
  ```

## 网络

- 所有服务通过 `app-network` 网络互通

## Web 管理界面

以下服务提供 Web 图形化管理界面，启动后可直接在浏览器中访问：

| 服务 | 访问地址 | 说明 |
|------|----------|------|
| Nacos 控制台 | [http://localhost:10000](http://localhost:10000) | 服务发现与配置管理控制台，默认无登录（若开启认证则需配置 Token） |
| Kafka UI | [http://localhost:10001](http://localhost:10001) | Kafka 集群管理界面，可查看 Topic、消息、消费者组等 |

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
```

### Nacos

> Nacos 控制台地址：[http://localhost:10000](http://localhost:10000)

```bash
docker run -d \
  --name nacos-server \
  -p 8848:8848 \
  -p 9848:9848 \
  -p 10000:8080 \
  -e MODE=standalone \
  -e SPRING_DATASOURCE_PLATFORM=derby \
  -e NACOS_AUTH_ENABLE=true \
  -e NACOS_AUTH_TOKEN=VGhpc0lzTXlTZWNyZXRLZXlGb3JOYWNvc0F1dGgyMDI1 \
  -e JVM_XMS=512m \
  -e JVM_XMX=512m \
  -e JVM_XMN=256m \
  nacos/nacos-server:latest
```

### Kafka（KRaft 模式）

> Kafka 在此使用 **KRaft 模式**，不依赖 Zookeeper，由 Kafka 自身管理集群元数据。
>
> Kafka UI 管理界面：[http://localhost:10001](http://localhost:10001)

```bash
docker run -d \
  --name kafka \
  -p 9092:9092 \
  -p 29092:29092 \
  -e CLUSTER_ID=MkU3OEVBNTcwNTJENDM2Qk \
  -e KAFKA_PROCESS_ROLES=broker,controller \
  -e KAFKA_NODE_ID=1 \
  -e KAFKA_CONTROLLER_QUORUM_VOTERS=1@kafka:29093 \
  -e KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER \
  -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:29092,CONTROLLER://0.0.0.0:29093,PLAINTEXT_HOST://0.0.0.0:9092 \
  -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092 \
  -e KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT \
  -e KAFKA_INTER_BROKER_LISTENER_NAME=PLAINTEXT \
  -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
  -e KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR=1 \
  -e KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS=0 \
  -e KAFKA_AUTO_CREATE_TOPICS_ENABLE=true \
  -e KAFKA_LOG_RETENTION_HOURS=24 \
  confluentinc/cp-kafka:7.6.1
```

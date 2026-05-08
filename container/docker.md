---
title: Docker 命令速查与部署指南
category: container
date: 2026-05-09
summary: Docker 命令速查与实战部署，涵盖镜像、容器、网络、卷管理及 Docker Compose。
---

## 镜像源配置

```json
"registry-mirrors": [
  "https://docker.mirrors.ustc.edu.cn",
  "https://registry.docker-cn.com",
  "http://hub-mirror.c.163.com",
  "https://mirror.ccs.tencentyun.com"
]
```

## 命令参考

### 1. 下载镜像

```bash
docker pull 镜像名:标签
```

从仓库拉取指定版本的镜像。省略标签则默认拉取 `latest`。

```bash
docker pull nginx:1.25
```

### 2. 运行容器

```bash
docker run [参数] 镜像名 [命令]
```

基于镜像创建并启动一个容器。

- 最简示例：`docker run nginx`

### 3. 挂载卷

```bash
docker run -v 宿主机路径:容器路径[:权限] 镜像
```

将宿主机目录或文件映射到容器内。权限默认可读写，加 `:ro` 为只读。

```bash
docker run -v /home/user/data:/app/data:ro nginx
```

> 也可用命名卷：`-v 卷名:容器路径`，先创建卷：`docker volume create 卷名`

### 4. run 其他常用参数

- `-d`：后台运行容器，打印容器 ID
- `-p 宿主机端口:容器端口`：端口映射
- `-e 变量名=值`：设置环境变量
- `--env-file 文件`：从文件加载环境变量
- `--name 名字`：指定容器名称
- `--network 网络名`：指定网络
- `--restart always`：容器退出时总是重启
- `--rm`：容器退出后自动删除
- `-w 目录`：指定工作目录
- `--entrypoint`：覆盖镜像默认入口
- `-h 主机名`：设置容器主机名
- `--dns`：指定 DNS 服务器
- `--ip`：指定容器 IP（需自定义网络）
- `-m/--memory`：内存限制（如 `512m`）
- `--cpus`：CPU配额（如 `1.5`）

```bash
docker run -d -p 8080:80 --name web -v ./html:/usr/share/nginx/html -e FOO=bar nginx
```

### 5. 调试容器与管理命令

- 查看运行中容器

  ```bash
  docker ps        # 运行中
  docker ps -a     # 全部容器
  ```

- 查看镜像

  ```bash
  docker images
  ```

- 查看日志

  ```bash
  docker logs 容器名/ID
  ```

  常用 `-f` 持续跟踪，`--tail 100` 最后100行，`--since 5m` 最近5分钟。

- 进入容器执行命令

  ```bash
  docker exec -it 容器名/ID /bin/bash
  ```

  若没有 bash 可用 sh：`/bin/sh`
  `-i` 交互，`-t` 分配伪终端。

- 直接启动容器并进入

  ```bash
  docker run -it --rm 镜像 /bin/bash
  ```

- 查看容器详情

  ```bash
  docker inspect 容器名/ID
  ```

- 查看容器内进程

  ```bash
  docker top 容器名
  ```

- 查看容器资源使用

  ```bash
  docker stats
  ```

- 从宿主机拷贝文件到容器

  ```bash
  docker cp 宿主机路径 容器名:容器路径
  ```

- 从容器拷贝文件到宿主机

  ```bash
  docker cp 容器名:容器路径 宿主机路径
  ```

- 暂停/恢复容器

  ```bash
  docker pause/unpause 容器名
  ```

- 停止/启动/重启容器

  ```bash
  docker stop/start/restart 容器名
  ```

- 删除容器

  ```bash
  docker rm 容器名    # 添加 -f 强制删除运行中容器
  ```

- 删除镜像

  ```bash
  docker rmi 镜像名:标签
  ```

- 清理无用资源

  ```bash
  docker container prune   # 清理已停止的容器
  docker image prune      # 清理悬空镜像
  docker volume prune    # 清理未使用的卷
  docker network prune  # 清理未使用的网络
  docker system prune -a   # 全面清理（包括未被任何容器使用的镜像）
  ```

- 容器端口查看

  ```bash
  docker port 容器名
  ```

### 6. 镜像与仓库

```bash
# 列出本地镜像
docker images

# 为镜像打标签
docker tag 源镜像:tag 新镜像:tag

# 推送镜像到仓库
docker push 镜像名:标签

# 登录/登出仓库
docker login [仓库地址]
docker logout

# 查看镜像历史/构成
docker history 镜像名:标签
```

### 7. 卷管理

```bash
docker volume ls                    # 列出卷
docker volume create 卷名           # 创建卷
docker volume inspect 卷名          # 查看卷详情
docker volume rm 卷名             # 删除卷（未使用）
```

挂载选项：

- 绑定挂载：`-v /host/path:/container/path`
- 命名卷：`-v myvolume:/container/path`
- 匿名卷：`-v /container/path`
- 只读挂载：`-v /host/path:/container/path:ro`

### 8. 网络管理

```bash
docker network ls                   # 列出网络
docker network create 网络名          # 创建网络（默认 bridge）
docker network inspect 网络名       # 查看网络详情
docker network connect 网络名 容器 # 将容器接入网络
docker network disconnect 网络名 容器
docker network rm 网络名           # 删除网络
```

### 9. 构建镜像脚本 (Dockerfile)

常用指令：

- `FROM 基础镜像`：指定基础镜像
- `RUN 命令`：在构建时执行命令
- `COPY 源 目标`：从构建上下文复制文件到镜像
- `ADD 源 目标`：类似COPY，但支持URL和自动解压tar
- `WORKDIR 路径`：设置工作目录
- `CMD ["可执行","参数"]`：容器默认运行命令（只有最后一个 CMD 生效）
- `ENTRYPOINT`：容器入口，不易被覆盖，常与 CMD 搭配
- `ENV 变量=值`：设定环境变量
- `EXPOSE 端口`：声明暴露端口（文档作用）
- `VOLUME ["路径"]`：声明匿名卷挂载点
- `USER 用户`：切换运行用户
- `ARG 变量=默认值`：构建参数（可用 --build-arg 传递）
- `HEALTHCHECK`：健康检查

示例 Dockerfile：

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
ENV NODE_ENV=production
EXPOSE 3000
USER node
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "server.js"]
```

构建命令：

```bash
docker build -t 镜像名:标签 -f Dockerfile路径 上下文路径
# 例：
docker build -t myapp:1.0 .
```

建议在构建目录添加 `.dockerignore` 文件，排除不需要的文件/目录（如 `node_modules`, `.git`），可加快构建并减小镜像体积。

### 10. Docker 网络模式

查看网络：`docker network ls`

- **bridge**（默认）：容器连接虚拟网桥，自动分配 IP，通过端口映射访问。

  ```bash
  docker run --network bridge ...
  ```

- **host**：容器与宿主机共享网络空间，无隔离，高效。

  ```bash
  docker run --network host ...
  ```

- **none**：无网络，容器只有 lo 回环接口。

  ```bash
  docker run --network none ...
  ```

- **自定义网络**（推荐）：支持容器间DNS解析（通过容器名互访）。

  ```bash
  docker network create mynet
  docker run --network mynet --name web nginx
  docker run --network mynet --name app myapp
  ```

  联网后，`web`容器内可直接`ping app`。

### 11. Docker Compose 写法

使用 `docker-compose.yml` 文件管理多容器应用（Compose v2 中 `version` 可省略，若保留推荐 "3.9" 以上）。

常用顶级元素：

- `services`：定义服务（容器）
- `networks`：定义网络
- `volumes`：定义命名卷

服务下常用配置：

| 键 | 含义 |
|---|---|
| `image` | 使用镜像 |
| `build` | 从Dockerfile构建（可指定上下文和dockerfile路径） |
| `ports` | 端口映射 `"宿主机:容器"` |
| `volumes` | 卷挂载，可用相对路径绑定挂载 `./data:/app/data` 或命名卷 |
| `environment` | 环境变量 (键值或列表) |
| `env_file` | 从文件加载环境变量 |
| `depends_on` | 控制服务启动顺序（不确保完全就绪，可配合健康检查） |
| `networks` | 指定加入的网络 |
| `command` | 覆盖默认命令 |
| `restart` | 重启策略，如 `always` |
| `healthcheck` | 健康检查配置 |
| `privileged` | 特权模式（谨慎使用） |
| `cap_add` / `cap_drop` | 添加/移除 Linux 能力 |
| `dns` | 自定义 DNS |

示例 docker-compose.yml：

```yaml
services:
  web:
    image: nginx:1.25
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
    networks:
      - front
    restart: always

  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    environment:
      - DB_HOST=db
    env_file:
      - ./api/.env.prod
    ports:
      - "3000:3000"
    networks:
      - front
      - back
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 5

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - back
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 5s
      timeout: 3s
      retries: 5

networks:
  front:
  back:

volumes:
  pgdata:
```

常用命令：

- 启动所有服务（后台）：`docker compose up -d`
- 停止并删除容器、网络：`docker compose down`
- 删除时同时删除匿名卷（加 `-v` 包括命名卷）：`docker compose down -v`
- 构建/重新构建服务：`docker compose build` 或 `up --build`
- 查看日志：`docker compose logs -f`
- 执行单次命令：`docker compose run --rm api npm test`
- 列出服务状态：`docker compose ps`
- 查看资源使用：`docker compose top`
- 拉取服务镜像：`docker compose pull`
- 暂停/恢复服务：`docker compose pause / unpause`

---

## 应用部署示例

### 通用应用

```bash
# 构建镜像
docker build -t ddd-user-web:2022.0.0 .

# 启动多个实例
docker run -d --name app1 --network mynet --network-alias app1 -p 8081:8080 ddd-user-web:2022.0.0
docker run -d --name app2 --network mynet --network-alias app2 -p 8082:8080 ddd-user-web:2022.0.0
docker run -d --name app3 --network mynet --network-alias app3 -p 8083:8080 ddd-user-web:2022.0.0
```

### MySQL

```bash
docker run -d --name mysql --network mynet --network-alias mysql \
  -p 3306:3306 \
  --restart always \
  -e TZ=Asia/Shanghai \
  -e MYSQL_ROOT_PASSWORD=123456 \
  mysql
```

### Canal

```bash
docker run -d --name canal --network mynet --network-alias canal \
  -p 11111:11111 \
  canal/canal-server
```

MySQL 添加 canal 用户：

```sql
CREATE USER canal IDENTIFIED WITH mysql_native_password BY 'canal';
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
FLUSH PRIVILEGES;
```

修改 canal 实例配置（挂载或进入容器）：

```bash
# 假设配置目录已挂载
vi canal-server/conf/example/instance.properties
# canal.master.position=
```

### Redis

单机部署：

```bash
docker run -d --name redis --network mynet --network-alias redis \
  -p 6379:6379 \
  --restart always \
  redis redis-server --appendfsync everysec
```

集群部署（IP 为宿主机物理地址，替换 `x.x.x.x`）：

```bash
docker run -d -p 6371:6379 -p 16371:16379 --name redis1 --network mynet --network-alias redis1 \
  redis redis-server --cluster-enabled yes --cluster-config-file nodes.conf \
  --cluster-announce-ip x.x.x.x --cluster-announce-port 6371 --cluster-announce-bus-port 16371 --appendfsync everysec

docker run -d -p 6372:6379 -p 16372:16379 --name redis2 --network mynet --network-alias redis2 \
  redis redis-server --cluster-enabled yes --cluster-config-file nodes.conf \
  --cluster-announce-ip x.x.x.x --cluster-announce-port 6372 --cluster-announce-bus-port 16372 --appendfsync everysec

docker run -d -p 6373:6379 -p 16373:16379 --name redis3 --network mynet --network-alias redis3 \
  redis redis-server --cluster-enabled yes --cluster-config-file nodes.conf \
  --cluster-announce-ip x.x.x.x --cluster-announce-port 6373 --cluster-announce-bus-port 16373 --appendfsync everysec

# 继续添加 redis4、redis5、redis6 直至 6 个节点...
```

```bash
# 创建集群（在任一节点内或安装了 redis-cli 的环境执行）
redis-cli --cluster create \
  x.x.x.x:6371 x.x.x.x:6372 x.x.x.x:6373 \
  x.x.x.x:6374 x.x.x.x:6375 x.x.x.x:6376 \
  --cluster-replicas 1
```

### Zookeeper

单机部署：

```bash
docker run -d --name zookeeper --network mynet --network-alias zookeeper \
  -p 6181:2181 \
  --restart always \
  zookeeper
```

集群部署：

```bash
docker-compose -f zk-cluster-compose.yml up -d
```

```yaml
# zk-cluster-compose.yml
services:
  zoo1:
    image: zookeeper
    restart: always
    hostname: zoo1
    container_name: zoo1
    ports:
      - "2181:2181"
    networks:
      - mynet
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181 server.4=zoo4:2888:3888;2181 server.5=zoo5:2888:3888;2181
  zoo2:
    image: zookeeper
    restart: always
    hostname: zoo2
    container_name: zoo2
    ports:
      - "2182:2181"
    networks:
      - mynet
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181 server.4=zoo4:2888:3888;2181 server.5=zoo5:2888:3888;2181
  zoo3:
    image: zookeeper
    restart: always
    hostname: zoo3
    container_name: zoo3
    ports:
      - "2183:2181"
    networks:
      - mynet
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181 server.4=zoo4:2888:3888;2181 server.5=zoo5:2888:3888;2181
  # zoo4、zoo5 配置类似...

networks:
  mynet:
    external: true
```

### Elasticsearch

```bash
docker run -d --name elasticsearch --network mynet --network-alias elasticsearch \
  -p 9200:9200 \
  -e "discovery.type=single-node" \
  -e "ES_JAVA_OPTS=-Xms2g -Xmx2g" \
  --restart always \
  elasticsearch:7.16.2
```

安装中文分词器（进入容器或挂载目录后执行）：

```bash
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.16.2/elasticsearch-analysis-ik-7.16.2.zip
```

### Kafka

```bash
# 注意：KAFKA_CFG_ADVERTISED_LISTENERS 需配置为宿主机实际IP，用于外部访问
docker run -d --name kafka --network mynet --network-alias kafka \
  -p 9092:9092 \
  -e ALLOW_PLAINTEXT_LISTENER=yes \
  -e KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181 \
  -e KAFKA_BROKER_ID=0 \
  -e KAFKA_CFG_LISTENERS=PLAINTEXT://:9092 \
  -e KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://<宿主机IP>:9092 \
  bitnami/kafka
```

### SkyWalking

docker-compose 部署（内置 H2 数据库）：

```yaml
# skywalking-compose.yml
services:
  skywalking-oap:
    image: apache/skywalking-oap-server
    container_name: skywalking-oap
    restart: always
    ports:
      - "11800:11800"
      - "12800:12800"
    environment:
      TZ: Asia/Shanghai

  skywalking-ui:
    image: apache/skywalking-ui
    container_name: skywalking-ui
    depends_on:
      - skywalking-oap
    restart: always
    ports:
      - "8888:8080"
    environment:
      SW_OAP_ADDRESS: skywalking-oap:12800
      TZ: Asia/Shanghai
```

```bash
docker-compose -f skywalking-compose.yml up -d
```

应用接入：下载 Java Agent，在应用启动参数中添加：

```bash
-javaagent:${path}/skywalking-agent.jar \
-Dskywalking.agent.service_name=${app-name} \
-Dskywalking.collector.backend_service=localhost:11800
```

### Nacos

docker-compose 快速部署（示例使用 Derby 单机）：

```bash
docker-compose -f standalone-derby.yaml up -d
```

控制台地址：http://127.0.0.1:8848/nacos/

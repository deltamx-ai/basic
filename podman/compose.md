# PostgreSQL (pgvector) + Redis Podman Compose 使用指南

本文档用于说明如何通过 **Podman** 部署与管理包含 `pgvector`（PostgreSQL 18 向量数据库）和 `Redis 8` 的编排服务，并附带配置文件的逐行解析。

---

## 一、快速上手（准备配置）

### 1. 创建项目目录

建议为服务创建独立目录（Compose 默认以当前文件夹名作为网络和容器命名前缀）：

```bash
mkdir deskpilot-db && cd deskpilot-db
```

### 2. 保存编排文件

将配置保存为 `docker-compose.yml`（或 `compose.yaml`；若使用较老版本的 `podman-compose`，`docker-compose.yml` 兼容性最好）：

```yaml
services:
  postgres:
    image: pgvector/pgvector:0.8.7-pg18-bookworm
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-deskpilot}
      POSTGRES_USER: ${POSTGRES_USER:-deskpilot}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD in .env}
    ports:
      - '127.0.0.1:${POSTGRES_PORT:-5432}:5432'
    volumes:
      - postgres_data:/var/lib/postgresql
    healthcheck:
      test:
        - CMD-SHELL
        - pg_isready -U ${POSTGRES_USER:-deskpilot} -d ${POSTGRES_DB:-deskpilot}
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 10s

  redis:
    image: redis:8-alpine
    restart: unless-stopped
    command:
      - redis-server
      - --appendonly
      - 'yes'
    ports:
      - '127.0.0.1:${REDIS_PORT:-6379}:6379'
    volumes:
      - redis_data:/data
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  postgres_data:
  redis_data:
```

### 3. 创建 `.env` 环境变量文件

配置中将 `POSTGRES_PASSWORD` 设为了必填项，请在**同级目录**下新建 `.env` 文件：

```env
# 【必填】数据库密码（不填会报错拒绝启动）
POSTGRES_PASSWORD=YourStrongPassword123

# 【选填】不写则使用 YAML 中的默认值
# POSTGRES_DB=deskpilot
# POSTGRES_USER=deskpilot
# POSTGRES_PORT=5432
# REDIS_PORT=6379
```

> ⚠️ **安全提示**：如果使用 Git 管理项目，请务必将 `.env` 添加到 `.gitignore` 中，避免泄露密码。

---

## 二、Podman 使用前注意事项

1. **确认 Compose 提供程序**：
   `podman compose` 底层依赖 `podman-compose` 或 `docker-compose`。若提示找不到命令，请先安装 `podman-compose`（如 `pip install podman-compose`、`brew install podman-compose` 或 `apt/dnf install podman-compose`）。
   *(注：如果你习惯直接运行 `podman-compose`，将下文命令开头的 `podman compose` 替换为 `podman-compose` 即可。)*

2. **macOS / Windows 用户需先启动虚拟机**：

   ```bash
   podman machine init   # 仅首次需要
   podman machine start  # 每次开机后需要运行
   ```

3. **解决“短镜像名”拉取报错（常见坑）**：
   部分 Linux 发行版下，Podman 后台静默拉取未带仓库前缀的镜像时可能会报 `short-name resolution enforced` 错误。可通过以下**任意一种方式**解决：
   * **方式一（推荐）**：将 YAML 中的 `image:` 补全完整仓库地址：
     * `docker.io/pgvector/pgvector:0.8.7-pg18-bookworm`
     * `docker.io/library/redis:8-alpine`
   * **方式二**：在终端先手动拉取一次（交互提示时选择 `docker.io`）：

     ```bash
     podman pull pgvector/pgvector:0.8.7-pg18-bookworm
     podman pull redis:8-alpine
     ```

4. **关于开机自启 (`restart: unless-stopped`)**：
   Podman 无后台常驻守护进程（Daemonless），Linux 主机重启后仅靠 YAML 中的 `unless-stopped` 不会自动拉起容器。如需生产环境开机自启，需配合 Podman **Quadlet** 或 `systemd` 托管。

---

## 三、常用管理命令速查

> 💡 **前提**：以下命令均需先 `cd` 进入存放 `docker-compose.yml` 和 `.env` 的目录执行。

### 1. 启动服务

```bash
# 创建并后台启动所有服务
podman compose up -d

# 仅启动单个服务（例如只启动 postgres）
podman compose up -d postgres
```

### 2. 查看状态与日志

```bash
# 查看容器运行状态、端口及健康检查状态（healthy）
podman compose ps

# 实时查看所有服务日志（Ctrl+C 退出，不影响容器运行）
podman compose logs -f

# 只看 postgres 最近 100 行日志并持续跟踪
podman compose logs -f --tail 100 postgres
```

### 3. 停止与恢复

```bash
# 【方式 A】仅暂停容器（保留容器，下次启动极快）
podman compose stop
podman compose stop redis      # 仅暂停 redis
podman compose start           # 重新启动已暂停的容器

# 【方式 B】停止并移除容器与网络（数据保留在命名卷中，不会丢失）
podman compose down
```

### 4. 重启与配置更新

```bash
# 重启服务
podman compose restart
podman compose restart postgres

# 修改了 .env 或 docker-compose.yml 后，必须重建容器才能生效（restart 不会重新加载配置）
podman compose up -d
```

### 5. 进入容器调试 / 验证连通性

```bash
# 连接进 PostgreSQL 命令行
podman compose exec postgres psql -U deskpilot -d deskpilot

# 测试 Redis 连通性（返回 PONG 表示正常）
podman compose exec redis redis-cli ping
```

### 6. 更新镜像与彻底销毁

```bash
# 拉取最新镜像并平滑重建服务
podman compose pull
podman compose up -d

# ⚠️【危险操作】停止容器并连同数据卷一起删除（会清空所有数据库和 Redis 数据！）
podman compose down -v
```

---

## 四、配置文件逐行解读

### 1. 顶级配置与 `postgres` 服务

| 配置行 | 说明 |
| :--- | :--- |
| `services:` | 顶级配置项，定义需要编排运行的容器服务列表。 |
| `postgres:` | 定义服务名为 `postgres`。同网络下其他容器可直接用主机名 `postgres` 访问它。 |
| `image: pgvector/pgvector:0.8.7-pg18-bookworm` | 使用集成 `pgvector 0.8.7`（向量检索扩展）的 PostgreSQL 18 镜像，底层为 Debian 12 (Bookworm)。 |
| `restart: unless-stopped` | 除非手动停止，否则容器异常退出时自动重启。 |
| `environment:` | 传入容器内部的初始化环境变量。 |
| `POSTGRES_DB: ${POSTGRES_DB:-deskpilot}` | 初始化的数据库名；未设置该环境变量时默认为 `deskpilot`。 |
| `POSTGRES_USER: ${POSTGRES_USER:-deskpilot}` | 数据库管理员用户名；未设置时默认为 `deskpilot`。 |
| `POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD in .env}` | 数据库密码；`:?` 语法表示**必填项**，未设置则直接报错中断并输出提示信息。 |
| `ports:` | 端口映射配置。 |
| `- '127.0.0.1:${POSTGRES_PORT:-5432}:5432'` | 仅绑定宿主机本地回环 `127.0.0.1`（防外网直接暴露），默认将本机 `5432` 映射到容器 `5432`。 |
| `volumes:` | 数据持久化挂载配置。 |
| `- postgres_data:/var/lib/postgresql` | 挂载命名卷 `postgres_data`。**注**：PostgreSQL 18 官方镜像将默认挂载点改为了 `/var/lib/postgresql`（而非旧版的 `/var/lib/postgresql/data`）。 |
| `healthcheck:` | 健康检查，判断数据库是否真正就绪可连接。 |
| `test:` | 指定健康检查命令。 |
| `- CMD-SHELL` | 通过容器内 `/bin/sh -c` 执行下一行命令。 |
| `- pg_isready -U ${POSTGRES_USER:-deskpilot} -d ${POSTGRES_DB:-deskpilot}` | 使用 `pg_isready` 探测指定用户和数据库是否已准备就绪。 |
| `interval: 5s` | 每 5 秒检查一次。 |
| `timeout: 5s` | 单次检查超时时间 5 秒。 |
| `retries: 10` | 连续失败 10 次标记为 `unhealthy`。 |
| `start_period: 10s` | 启动宽限期 10 秒，期间初始化失败不计入 `retries`。 |

### 2. `redis` 服务

| 配置行 | 说明 |
| :--- | :--- |
| `redis:` | 定义第二个服务，服务名为 `redis`。 |
| `image: redis:8-alpine` | 使用基于轻量级 Alpine Linux 构建的 Redis 8 官方镜像。 |
| `restart: unless-stopped` | 除非手动停止，否则自动重启。 |
| `command:` | 覆盖镜像默认启动命令。 |
| `- redis-server` | 启动 Redis 服务端。 |
| `- --appendonly` | 传递持久化开关参数。 |
| `- 'yes'` | 开启 AOF 持久化（加引号防止 YAML 将无引号的 `yes` 解析为布尔值 `true`）。 |
| `ports:` | 端口映射配置。 |
| `- '127.0.0.1:${REDIS_PORT:-6379}:6379'` | 仅允许本机 `127.0.0.1` 访问，默认将宿主机 `6379` 映射到容器 `6379`。 |
| `volumes:` | 数据持久化挂载配置。 |
| `- redis_data:/data` | 将命名卷 `redis_data` 挂载到容器 `/data`（Redis 持久化文件默认存放目录）。 |
| `healthcheck:` | 健康检查配置。 |
| `test: ['CMD', 'redis-cli', 'ping']` | 直接执行 `redis-cli ping`，返回 `PONG` 即表示健康。 |
| `interval: 5s` | 每 5 秒检查一次。 |
| `timeout: 3s` | 单次检查超时时间 3 秒。 |
| `retries: 10` | 连续失败 10 次标记为 `unhealthy`。 |

### 3. 数据卷声明

| 配置行 | 说明 |
| :--- | :--- |
| `volumes:` | 顶级配置项，声明交由容器引擎统一管理的命名数据卷（Named Volumes）。 |
| `postgres_data:` | 声明 PostgreSQL 持久化卷（使用默认本地驱动）。 |
| `redis_data:` | 声明 Redis 持久化卷（使用默认本地驱动）。 |

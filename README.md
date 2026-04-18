# arT-Tool-docker

使用预构建镜像运行 [CleanSheet Web](https://github.com/arT-bai/CleanSheetWeb)（Rust Web 服务），**无需本地编译**。

## 镜像

托管在 GitHub Container Registry。Docker 要求镜像路径为小写，实际拉取地址为：

`ghcr.io/art-bai/art-tool-docker`

（与 GitHub 上的 `arT-bai/arT-Tool-docker` 仓库对应。）

## 前置条件

- 已安装 [Docker](https://docs.docker.com/get-docker/) 与 [Docker Compose](https://docs.docker.com/compose/install/)（Compose V2：`docker compose`）。

## 快速启动

```bash
git clone https://github.com/arT-bai/arT-Tool-docker.git
cd arT-Tool-docker
docker compose up -d
```

浏览器访问：<http://localhost:2000>

## 使用说明

### Web 访问

- 首页：`http://localhost:2000/`
- 健康检查：`http://localhost:2000/health`

### API 路由

- `POST /preview`：预览清洗结果（不落盘最终文件）
- `POST /clean`：执行清洗并生成可下载结果
- `GET /download/:job_id`：下载生成文件

### 常见运维命令

```bash
# 查看运行状态
docker compose ps

# 查看日志
docker compose logs -f

# 停止服务
docker compose down

# 停止并清理数据卷（谨慎）
docker compose down -v
```

## 仅使用 Docker（不克隆本仓库）

```bash
docker pull ghcr.io/art-bai/art-tool-docker:latest
docker run -d --name art-tool-docker -p 2000:2000 \
  -v art-tool-storage:/app/storage \
  --restart unless-stopped \
  ghcr.io/art-bai/art-tool-docker:latest
```

## 数据持久化

默认 Compose 使用命名卷 `art-tool-storage`，挂载到容器内 `/app/storage`（上传与输出目录）。

若需将数据放在本机目录，可改为：

```yaml
volumes:
  - ./storage:/app/storage
```

## 项目框架

当前服务基于 Rust + Axum，运行时核心目录如下（对应 [CleanSheetWeb](https://github.com/arT-bai/CleanSheetWeb) 源码）：

```text
CleanSheetWeb/
├─ src/
│  ├─ main.rs             # 应用入口：启动 HTTP 服务、初始化日志与定时清理任务
│  ├─ config.rs           # AppConfig 默认配置（端口、上传目录、TTL 等）
│  ├─ state.rs            # 全局共享状态（配置 + JobStore）
│  ├─ routes/             # 路由注册（/clean /preview /download/:job_id）
│  ├─ handlers/           # HTTP 处理器（参数解析、响应组装）
│  ├─ services/           # 业务逻辑（加载、匹配、清洗、导出、任务存储）
│  ├─ domain/             # 领域模型与 DTO
│  └─ utils/              # 文件与文本辅助工具
├─ templates/             # Askama HTML 模板
├─ assets/                # 静态资源（前端样式/脚本）
├─ storage/               # 运行期数据目录（uploads / outputs）
├─ Dockerfile             # 多阶段构建镜像
└─ docker-compose.yml     # 本地/服务器启动编排
```

### 运行机制（简述）

- 容器启动后监听 `0.0.0.0:2000`。
- 上传文件与清洗结果写入 `/app/storage`（通过卷映射持久化）。
- 服务包含周期性清理任务，会移除过期作业与历史文件。

## 镜像构建说明

镜像由源码仓库 CI 构建并推送，见 [CleanSheetWeb · GitHub Actions](https://github.com/arT-bai/CleanSheetWeb/actions)（若该仓库已启用工作流）。

## 版本标签

- `latest`：默认分支最新构建。
- `v*`（如 `v0.1.0`）：对应 Git 标签的语义化版本。
- `sha-xxxxxxx`：具体构建提交。

拉取指定版本示例：

```bash
docker pull ghcr.io/art-bai/art-tool-docker:v0.1.0
```

将 `docker-compose.yml` 中的 `image` 改为同一标签即可。

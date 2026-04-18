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

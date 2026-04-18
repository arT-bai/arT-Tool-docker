# arT-Tool-docker

> 一个基于 Rust 的轻量数据清洗 Web 应用：上传表格、预览数据、按字段清洗前 N 行，并导出新的 `.xlsx` 文件。

## 目录

- [项目简介](#项目简介)
- [功能亮点](#功能亮点)
- [技术栈](#技术栈)
- [快速开始](#快速开始)
- [Docker 与离线部署](#docker-与离线部署)
- [核心接口](#核心接口)
- [API 示例](#api-示例)
- [项目结构](#项目结构)
- [安全与边界](#安全与边界)
- [清理机制](#清理机制)
- [测试建议](#测试建议)
- [后续规划](#后续规划)

## 项目简介

CleanSheet Web 面向“表格数据快速清洗”场景，提供一个开箱即用的 Web 页面：

- 上传 `csv/xls/xlsx`
- 解析并预览表头与样本行
- 输入多个字段名进行匹配
- 指定前 N 行执行清洗
- 生成并下载 `.xlsx` 清洗结果

适合内部工具、小团队数据处理和离线内网部署场景。

## 功能亮点

- 支持多格式导入：`csv` / `xls` / `xlsx`
- 预览体验友好：展示表头、总行数与前 10 行数据
- 字段匹配可视化：区分 `matched_fields` 与 `unmatched_fields`
- 可控清洗范围：仅处理前 N 行，避免误操作全量数据
- 导出稳定：统一导出为 `.xlsx`
- 错误处理完善：页面接口与下载接口均返回合理错误信息
- 基础安全防护：上传白名单、文件名校验、请求体大小限制
- 自动过期清理：上传文件、导出文件与任务元数据定期回收

## 技术栈

- **语言与运行时**: Rust 2021, Tokio
- **Web 框架**: Axum
- **页面渲染**: Askama + Vanilla JS(Fetch) + Pico CSS
- **文件处理**: `csv`, `calamine`, `rust_xlsxwriter`
- **日志与可观测性**: `tracing`, `tracing-subscriber`

## 快速开始

### 1) 环境准备

确保已安装 Docker（建议 Docker Desktop，含 Compose V2）：

```bash
docker --version
docker compose version
```

### 2) 启动方式

方式 A：使用本仓库 Compose（推荐）

```bash
git clone https://github.com/arT-bai/arT-Tool-docker.git
cd arT-Tool-docker
docker compose up -d
```

方式 B：仅 Docker 命令（不克隆仓库）

```bash
docker pull ghcr.io/art-bai/art-tool-docker:latest
docker run -d --name art-tool-docker -p 2000:2000 \
  -v art-tool-storage:/app/storage \
  --restart unless-stopped \
  ghcr.io/art-bai/art-tool-docker:latest
```

启动后访问：<http://localhost:2000>

## Docker 与离线部署

### 本地 Docker 运行

```bash
docker pull ghcr.io/art-bai/art-tool-docker:latest
docker run --rm -p 2000:2000 ghcr.io/art-bai/art-tool-docker:latest
```

或使用 Compose：

```bash
docker compose up -d
```

### 离线内网部署

可在联网机器先拉取镜像并导出：

```bash
docker pull ghcr.io/art-bai/art-tool-docker:latest
docker save -o art-tool-docker-latest.tar ghcr.io/art-bai/art-tool-docker:latest
```

在内网机器导入并运行：

```bash
docker load -i art-tool-docker-latest.tar
docker run --rm -p 2000:2000 ghcr.io/art-bai/art-tool-docker:latest
```

## 核心接口

- `GET /`：首页
- `POST /preview`：上传并预览
- `POST /clean`：执行清洗并导出
- `GET /download/:job_id`：下载结果文件

## API 示例

### `POST /preview`

- Content-Type: `multipart/form-data`
- 字段说明：
  - `file`: 上传文件（`.csv/.xls/.xlsx`）
  - `field_names`: 逗号分隔字段（例如 `Phone,Email`）
  - `row_limit`: 正整数（例如 `1000`）

成功响应示例：

```json
{
  "upload_token": "xxx",
  "original_filename": "demo.xlsx",
  "selected_fields": ["Phone", "Email"],
  "matched_fields": ["Phone", "Email"],
  "unmatched_fields": [],
  "row_limit": 1000,
  "total_rows": 1280,
  "headers": ["ID", "Name", "Phone", "Email"],
  "rows": [["1", "Tom", " 123 ", " a@b.com "]]
}
```

### `POST /clean`

- Content-Type: `application/json`

请求体示例：

```json
{
  "upload_token": "xxx",
  "field_names": ["Phone", "Email"],
  "row_limit": 1000
}
```

成功响应示例：

```json
{
  "output_filename": "demo_cleaned_20260418_123456.xlsx",
  "processed_rows": 1000,
  "matched_fields": ["Phone", "Email"],
  "download_link": "/download/xxx"
}
```

## 项目结构

```text
CleanSheetWeb/
├── src/
│   ├── handlers/      # HTTP 处理层（编排请求/响应）
│   ├── routes/        # 路由注册
│   ├── services/      # 核心业务（loader/matcher/cleaner/exporter/job_store）
│   ├── domain/        # 领域模型与 DTO
│   ├── utils/         # 工具函数（文本解析、文件保存）
│   ├── config.rs
│   ├── error.rs
│   ├── main.rs
│   └── state.rs
├── templates/         # Askama 模板与 partials
├── assets/            # 静态资源（CSS / 图片）
├── storage/
│   ├── uploads/
│   └── outputs/
├── scripts/
│   └── run-dev.sh
├── docs/
│   ├── codebook.md
│   └── docker-offline-deploy.md
```

## 安全与边界

- 上传扩展名白名单：仅允许 `.csv/.xls/.xlsx`
- 文件名安全校验：拒绝空文件名与目录穿越字符
- 请求体大小限制：`max_upload_mb`（见 `src/config.rs`）
- 错误处理策略：
  - 页面接口返回 JSON（如 `{ "message": "..." }`）
  - 下载接口返回语义化状态码（如 `404`）

## 清理机制

- `JobStore` 保存上传与导出任务元数据（含过期时间）
- 后台任务每 3 分钟执行一次：
  - 清理过期上传/导出文件
  - 清理过期任务记录
- 清理入口：`cleanup_expired_jobs(...)`

## 测试建议

建议按以下顺序手工验收：

1. 上传 CSV，验证预览和字段匹配
2. 上传 XLSX，验证预览和字段匹配
3. 设置字段 + `row_limit` 后执行清洗
4. 下载生成文件，确认可正常打开
5. 验证仅命中字段且仅前 N 行被清洗
6. 使用无效扩展名/超大文件验证错误提示
7. 调小 TTL 或等待过期，验证自动清理逻辑

## 后续规划

- 下载鉴权与访问令牌签名
- 更细粒度清洗规则配置
- 后台异步任务与进度反馈
- 历史任务列表与可观测性增强

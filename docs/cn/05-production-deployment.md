# LightRAG 生产部署：API 服务与 Web UI

> 💡 **学习目标**：
> - 掌握如何使用交互式向导快速生成生产级配置文件。
> - 学习启动自带的 FastAPI 服务端点，并了解多进程部署。
> - 了解如何编译和运行可视化的 Web UI 控制台。

## 1. 架构全景

在生产环境中，我们通常不直接运行 Python 脚本，而是将 LightRAG 封装为一个后端微服务。LightRAG 官方提供了一套完整的解决方案：
- **后端**：基于 FastAPI 构建，提供 RESTful 接口，并兼容 Ollama 聊天 API 格式（这意味着你可以将 LightRAG 直接无缝接入 Open WebUI 等开源前端）。
- **前端**：基于 React 19 + TypeScript 开发的现代 Web UI。

## 2. 交互式环境配置（Setup Wizard）

不要手动编写繁杂的 `.env` 文件。LightRAG 提供了一个基于 `Makefile` 的交互式配置向导，帮助你一步步生成安全的配置和 Docker 编排文件。

```bash
# 1. 基础配置（必须）
# 引导你配置 LLM、Embedding 模型、API Key 等核心参数
make env-base

# 2. 存储后端配置（可选但推荐）
# 引导你配置 PostgreSQL、Neo4j 等外部数据库，并自动生成 docker-compose.final.yml
make env-storage

# 3. 服务器与安全配置（可选）
# 配置 API 监听端口、CORS、SSL 证书以及鉴权
make env-server

# 4. 安全审计
# 检查当前生成的 .env 文件是否存在泄露风险或配置缺陷
make env-security-check
```

## 3. 编译 Web UI

前端 Web UI 位于 `lightrag_webui` 目录下，采用 `Bun` 和 `Vite` 构建。

```bash
# 进入前端目录
cd lightrag_webui

# 安装依赖（必须使用 bun，不要用 npm/yarn）
bun install --frozen-lockfile

# 构建生产环境的静态文件
bun run build

# 返回项目根目录
cd ..
```

构建完成后，编译产物会被后端 FastAPI 自动挂载并提供静态服务。

## 4. 启动服务

你有两种方式来启动最终的服务：

### 方式 A：直接在宿主机运行（适合开发/独立部署）

确保你的 `.env` 文件就绪，并且所需的数据库（如果有）已经启动。

```bash
# 启动带有 API 和 Web UI 的综合服务器 (单进程)
lightrag-server

# 开发热重载模式：
uvicorn lightrag.api.lightrag_server:app --reload

# 生产级高并发模式（使用 Gunicorn 管理多进程）：
lightrag-gunicorn
```

### 方式 B：使用 Docker Compose 运行（适合容器化生产环境）

如果你在 `make env-storage` 阶段选择了基于 Docker 的数据库部署，系统会生成一个组合好的 `docker-compose.final.yml`。

```bash
# 启动所有关联的服务（数据库、缓存、LightRAG API 容器等）
docker compose -f docker-compose.final.yml up -d
```

## 5. 验证部署

服务启动后：
1. **Web UI**：在浏览器中访问 `http://localhost:8020`（端口取决于你的 `.env` 配置），你将看到可视化的知识图谱管理界面，可在此直接上传文档和测试查询。
2. **API 文档**：访问 `http://localhost:8020/docs` 查看自动生成的 Swagger 接口文档，方便第三方系统（如自建的聊天机器人或企业微信后台）进行集成。

---
🎉 **结语**：至此，你已经完成了从 LightRAG 核心机制学习、模型调优、企业级存储配置，到最终完整应用上线的全部旅程！

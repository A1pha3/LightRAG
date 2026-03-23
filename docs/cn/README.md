# 📚 LightRAG 中文文档中心 (Chinese Documentation)

欢迎来到 LightRAG 官方中文文档库！无论你是刚接触图检索（Graph RAG）的初学者，还是准备在企业级环境中部署高可用架构的资深架构师，本系列文档都将为你提供全面、深度的指导。

> **💡 LightRAG 是什么？**
> LightRAG 是一个将**知识图谱（Knowledge Graph）**与**向量检索（Vector Retrieval）**深度融合的先进 RAG 框架。它通过大模型从文档中提取实体与逻辑关系，让你在面对“宏观总结”、“跨文档推理”等复杂问题时，获得传统向量检索无法企及的精准度和全局视野。

---

## 🗺️ 文档导航 (Learning Path)

我们精心设计了一条从入门到精通的学习路径，建议按顺序阅读：

### 🔰 第一阶段：新手起步 (Beginner)
**[01. 新手入门：快速上手指南](./01-getting-started.md)**
- 为什么需要 LightRAG？它解决了什么痛点？
- 本地开发环境的快速安装 (`uv` 构建)。
- 编写并运行你的第一个 LightRAG 混合查询脚本。

### ⚙️ 第二阶段：核心原理 (Intermediate)
**[02. 核心概念：架构与查询模式深度解析](./02-core-architecture.md)**
- 揭秘底层四大存储支柱（KV、向量、图、文档状态）。
- 详解 5 种查询模式（`naive`, `local`, `global`, `hybrid`, `mix`）的底层原理与最佳适用场景。

**[03. 模型集成：LLM、Embedding 与 Reranker](./03-custom-models.md)**
- 接入 Ollama 本地模型与云端 API（Azure、Gemini 等）。
- 攻克自定义 Embedding 函数的封装陷阱（`@wrap_embedding_func_with_attrs`）。
- 配置 Reranker（重排器）大幅提升 `mix` 模式的检索质量。

### 🚀 第三阶段：生产架构 (Expert)
**[04. 专家级存储：图数据库与企业级后端](./04-enterprise-storage.md)**
- 告别本地 JSON：如何部署并连接 Neo4j 获得极致的图遍历性能。
- 一体化大一统：使用 PostgreSQL（+ pgvector + AGE）处理所有存储需求。
- 多租户与数据隔离（Workspace 机制）。

**[05. 生产部署：API 服务与 Web UI](./05-production-deployment.md)**
- 使用交互式向导（Setup Wizard）自动生成安全配置。
- 使用 Bun 编译现代化可视化 Web UI。
- 基于 FastAPI 和 Docker Compose 的高并发后端服务部署。

---

## 💡 最佳实践与避坑速记

为了节省你的调试时间，请牢记以下开发法则：

1. **必须初始化**：在实例化 `LightRAG` 后，永远不要忘记调用 `await rag.initialize_storages()`。
2. **上下文窗口**：用于图谱提取的 LLM 必须具备大上下文能力。如果使用 Ollama，必须显式通过 `llm_model_kwargs` 将 `num_ctx` 设置为至少 `32768`。
3. **更换模型需清空数据**：一旦改变了 Embedding 模型，由于向量维度不匹配，**必须**清空原有的向量存储目录或数据库表。
4. **推荐的查询模式**：在配置了 Reranker 的生产环境中，强烈建议将 `mix` 作为默认查询模式。

---
*本系列文档由 `cn-doc-writer` AI 专家辅助构建。如果您在阅读过程中发现任何纰漏或需要补充的章节，欢迎在 GitHub 提交 Issue 或 PR！*

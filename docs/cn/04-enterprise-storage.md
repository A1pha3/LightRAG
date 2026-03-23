# LightRAG 专家级存储：图数据库与企业级后端

> 💡 **学习目标**：
> - 了解为什么在生产环境中需要替换默认的本地文件存储。
> - 掌握如何配置 Neo4j 图数据库。
> - 学习使用 PostgreSQL 打造统一的存储底座。
> - 掌握基于 Workspace 的数据隔离机制。

## 1. 为什么要升级存储后端？

默认情况下，LightRAG 使用本地文件（JSON、NetworkX 内存图、NanoVector 内存向量库）来存储数据。这种方式非常适合本地测试和开发，但在生产环境中存在以下致命缺陷：
- **内存受限**：大规模图谱（几十万个节点）会瞬间撑爆服务器内存。
- **不支持并发**：本地文件无法支持多个 LightRAG API 实例同时进行读写。
- **缺乏持久化保障**：进程崩溃可能导致数据损坏。

为此，LightRAG 提供了丰富的企业级后端支持，包括 PostgreSQL、MongoDB、Neo4j、Milvus、Redis 等。

## 2. 部署 Neo4j 图数据库（强推）

在所有的图存储后端中，**Neo4j 提供了最佳的生产级性能**。它能够以极快的速度执行复杂的图遍历和子图提取。

### 步骤 2.1：启动 Neo4j 服务

使用 Docker 快速启动一个 Neo4j 实例：

```bash
docker run -d \
    --name neo4j \
    -p 7474:7474 -p 7687:7687 \
    -e NEO4J_AUTH=neo4j/password \
    neo4j:latest
```

### 步骤 2.2：在代码中配置 LightRAG

通过环境变量或显式传参将图存储切换为 Neo4j：

```python
import os
import asyncio
from lightrag import LightRAG

# 配置 Neo4j 连接信息
os.environ["NEO4J_URI"] = "neo4j://localhost:7687"
os.environ["NEO4J_USERNAME"] = "neo4j"
os.environ["NEO4J_PASSWORD"] = "password"
os.environ["NEO4J_DATABASE"] = "neo4j" # 社区版默认数据库名

async def main():
    rag = LightRAG(
        working_dir="./rag_storage",
        # 显式指定图存储引擎为 Neo4j
        graph_storage="Neo4JStorage",
        # 向量和 KV 依然可以使用其他的，比如 PostgreSQL
        # vector_storage="PGVectorStorage",
        # kv_storage="PGKVStorage",
        # ... LLM 配置 ...
    )
    
    await rag.initialize_storages()
    print("成功连接到 Neo4j 图数据库！")
    await rag.finalize_storages()

if __name__ == "__main__":
    asyncio.run(main())
```

## 3. 大一统方案：PostgreSQL

如果你不希望维护多种数据库（同时维护 Neo4j、Milvus、Redis 会增加运维成本），你可以使用 PostgreSQL 作为**一体化存储方案**。

PostgreSQL 通过扩展插件，可以同时胜任四大存储角色：
- **KV 存储 / DocStatus 存储**：原生关系型表。
- **向量存储**：依赖 `pgvector` 插件。
- **图存储**：依赖 Apache AGE 插件。

> ⚠️ **注意**：你需要部署带有 `pgvector` 和 `age` 插件的 PostgreSQL（建议版本 >= 16.6）。

配置方式同样简单：

```python
rag = LightRAG(
    working_dir="./rag_storage",
    kv_storage="PGKVStorage",
    vector_storage="PGVectorStorage",
    graph_storage="PGGraphStorage",
    doc_status_storage="PGDocStatusStorage",
)
```
*前提是你在环境中配置了 `POSTGRES_URI` 环境变量。*

## 4. 数据隔离机制（Workspace）

在 SaaS 平台或多用户场景下，你可能希望不同的用户拥有各自独立的知识库，互不干扰。LightRAG 提供了 `workspace` 参数来实现这一点。

```python
rag_user_a = LightRAG(
    working_dir="./rag_storage",
    workspace="tenant_a_finance", # 租户 A 的财务知识库
    # ...
)

rag_user_b = LightRAG(
    working_dir="./rag_storage",
    workspace="tenant_b_hr",      # 租户 B 的人事知识库
    # ...
)
```

**底层实现原理**：
- **文件存储**：会在工作目录下创建名为 `tenant_a_finance` 的子目录。
- **关系型数据库（如 PostgreSQL）**：会在表中增加一个 `workspace` 列进行数据行级别的过滤。
- **文档数据库（如 MongoDB）**：会使用 `tenant_a_finance` 作为集合（Collection）的前缀。

掌握了存储引擎的配置后，你的系统已经具备了生产环境的承载能力。最后，请阅读 [生产部署：API 服务与 Web UI](./05-production-deployment.md) 来完成系统的最终上线。

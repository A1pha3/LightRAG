# LightRAG 专家级存储：图数据库与企业级后端

> 💡 **学习目标**：
> - 了解为什么在生产环境中需要替换默认的本地文件存储。
> - 掌握如何配置 Neo4j 图数据库，并了解其性能优势。
> - 学习使用 PostgreSQL 打造统一的存储底座。
> - 掌握基于 Workspace 的数据隔离机制与大规模调优技巧。

## 1. 为什么要升级存储后端？

默认情况下，LightRAG 使用本地文件（JSON、NetworkX 内存图、NanoVector 内存向量库）来存储数据。这种方式非常适合本地测试和开发，但在生产环境中存在以下致命缺陷：
- **内存受限**：大规模图谱（几十万个节点）会瞬间撑爆服务器内存。
- **不支持并发**：本地文件无法支持多个 LightRAG API 实例同时进行读写。
- **缺乏持久化保障**：进程崩溃可能导致数据损坏。

为此，LightRAG 提供了丰富的企业级后端支持，包括 PostgreSQL、MongoDB、Neo4j、Milvus、Redis、OpenSearch 等。

## 2. 部署 Neo4j 图数据库（强推）

在所有的图存储后端中，**Neo4j 提供了最佳的生产级性能**。它能够以极快的速度执行复杂的图遍历和子图提取。根据官方测试，在生产环境中，Neo4j 的性能显著优于带有 AGE 插件的 PostgreSQL。

### 步骤 2.1：启动 Neo4j 服务

使用 Docker 快速启动一个 Neo4j 实例（社区版）：

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
        # 向量和 KV 依然可以使用其他的，比如 PostgreSQL 或 Milvus
        # vector_storage="MilvusVectorDBStorage",
        # kv_storage="RedisKVStorage",
        # ... LLM 配置 ...
    )
    
    await rag.initialize_storages()
    print("成功连接到 Neo4j 图数据库！")
    await rag.finalize_storages()

if __name__ == "__main__":
    asyncio.run(main())
```

## 3. 大一统方案：PostgreSQL 与 OpenSearch

如果你不希望维护多种数据库（同时维护 Neo4j、Milvus、Redis 会增加极大的运维成本），你可以使用单一数据库作为**一体化存储方案**。

### 方案 A：PostgreSQL
PostgreSQL 通过扩展插件，可以同时胜任四大存储角色：
- **KV 存储 / DocStatus 存储**：原生关系型表（`PGKVStorage`, `PGDocStatusStorage`）。
- **向量存储**：依赖 `pgvector` 插件（`PGVectorStorage`）。
- **图存储**：依赖 Apache AGE 插件（`PGGraphStorage`）。

> ⚠️ **注意**：你需要部署带有 `pgvector` 和 `age` 插件的 PostgreSQL（建议版本 >= 16.6）。

### 方案 B：OpenSearch (v2.12+)
OpenSearch 同样支持全量存储，特别适合已有 ELK 技术栈的团队：
- `OpenSearchKVStorage`
- `OpenSearchVectorDBStorage`
- `OpenSearchGraphStorage`
- `OpenSearchDocStatusStorage`

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

## 5. 专家级调优：大规模并发插入

在处理海量文档时，单线程插入会非常缓慢。你可以通过 `max_parallel_insert` 提升并发：

```python
rag = LightRAG(
    # ... 其他配置 ...
    max_parallel_insert=4 # 默认是 2
)

# 批量传入数组
await rag.ainsert(["文档1", "文档2", "文档3", "文档4", "文档5"])
```

> ⚠️ **性能瓶颈提示**：建议将 `max_parallel_insert` 控制在 10 以内。真正的瓶颈通常不在于 LightRAG 框架，而在于**你的 LLM API 的并发速率限制（Rate Limits）**。并发过高会导致 LLM 服务返回 HTTP 429 (Too Many Requests) 错误。

掌握了存储引擎的配置后，你的系统已经具备了生产环境的承载能力。最后，请阅读 [05. 生产部署：API 服务与 Web UI](./05-production-deployment.md) 来完成系统的最终上线。

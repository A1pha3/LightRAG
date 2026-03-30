# LightRAG 新手入门：快速上手指南

> 💡 **学习目标**：
> - 了解 LightRAG 解决的核心痛点（为什么需要它）。
> - 完成本地开发环境的安装，并了解离线部署的基本思路。
> - 编写并运行第一个完整的 LightRAG 程序，掌握 `insert` 与 `query` 的基础参数。

## 1. 为什么选择 LightRAG？

传统的检索增强生成（RAG）系统主要依赖向量相似度检索。这种方式在回答“特定事实”时表现良好，但面对需要跨文档关联、宏观总结（例如“这本书的主要主题是什么？”）的复杂问题时，往往会丢失全局上下文。

LightRAG 引入了**知识图谱（Knowledge Graph）**。在文档索引阶段，它会使用大语言模型（LLM）提取文本中的“实体（Entity）”和“关系（Relation）”，并构建出一张图。在查询阶段，它不仅检索相似的文本块，还会顺着图结构检索相关的实体和逻辑链路，从而让 AI 拥有“全局视野”。

## 2. 环境安装

我们强烈推荐使用 `uv` 来管理 Python 依赖，它比传统的 `pip` 更快且更可靠。

```bash
# 1. 克隆代码仓库
git clone https://github.com/HKUDS/LightRAG.git
cd LightRAG

# 2. 使用 uv 同步依赖并创建虚拟环境
uv sync

# 3. 激活虚拟环境
# macOS / Linux:
source .venv/bin/activate
# Windows:
# .venv\Scripts\activate

# 4. 复制环境变量模板（后续用于配置 API Key 和数据库连接）
cp env.example .env
```

> **📦 离线部署提示**：如果你处于无法连接外网的内网环境中，你需要提前在一台联网机器上下载好所有的 Python 依赖包和 HuggingFace 上的 Embedding 模型权重，然后将整个 `.venv` 目录和模型权重目录拷贝到内网机器中。

## 3. 你的第一个 LightRAG 程序

下面是一个完整可运行的示例。在这个例子中，我们将使用 OpenAI 的模型进行文档解析和查询。

> ⚠️ **关键注意**：在实例化 `LightRAG` 之后，**必须**使用 `await rag.initialize_storages()` 初始化存储后端，否则程序会抛出异常。这是最常见的初学者错误！

```python
import os
import asyncio
from lightrag import LightRAG, QueryParam
from lightrag.llm.openai import gpt_4o_mini_complete, openai_embed
from lightrag.utils import setup_logger

# 启用日志，级别设为 INFO，方便观察实体提取和检索的过程
setup_logger("lightrag", level="INFO")

# 确保配置了 OpenAI API Key（生产环境中建议通过 .env 文件加载）
os.environ["OPENAI_API_KEY"] = "sk-你的API密钥"

# 定义数据存储目录
WORKING_DIR = "./rag_storage"
if not os.path.exists(WORKING_DIR):
    os.mkdir(WORKING_DIR)

async def main():
    rag = None
    try:
        # 1. 初始化 LightRAG 实例
        rag = LightRAG(
            working_dir=WORKING_DIR,
            embedding_func=openai_embed,           # 用于将文本转换为向量
            llm_model_func=gpt_4o_mini_complete,   # 用于提取实体和生成最终回答
        )
        
        # 2. 强制要求：初始化底层存储！
        await rag.initialize_storages()
        
        # 3. 插入文档
        # 在此阶段，LLM 会在后台被调用，消耗 Token 来提取实体和关系。
        # 提示：你可以使用 batch 插入：await rag.ainsert(["文档1", "文档2"])
        document_text = "LightRAG 是香港大学数据科学实验室开源的先进检索系统。它结合了图谱和向量检索。"
        await rag.ainsert(document_text)

        # 4. 执行查询
        # 我们使用 hybrid（混合）模式，它会同时检索局部细节和全局上下文。
        # 也可以通过 top_k 参数限制检索回来的图谱节点数量。
        query_text = "LightRAG 是由谁开源的？它的核心技术是什么？"
        result = await rag.aquery(
            query_text,
            param=QueryParam(
                mode="hybrid",
                top_k=60  # 默认值通常为 60，表示最多检索 60 个相关实体/关系
            )
        )
        print("========== 查询结果 ==========")
        print(result)

    except Exception as e:
        print(f"运行出错: {e}")
    finally:
        # 5. 清理与持久化存储
        # 这一步非常重要，它确保所有在内存中排队的数据都被写入到磁盘或数据库中。
        if rag:
            await rag.finalize_storages()

if __name__ == "__main__":
    asyncio.run(main())
```

## 4. 下一步学习

恭喜！你已经成功运行了 LightRAG 并理解了基础的代码骨架。在真实的生产环境中，我们会有大量的文档和复杂的查询需求。接下来，请阅读 [02. 核心架构与查询模式深度解析](./02-core-architecture.md) 来深入了解底层机制。

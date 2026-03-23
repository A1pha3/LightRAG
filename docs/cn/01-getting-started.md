# LightRAG 新手入门：快速上手指南

> 💡 **学习目标**：
> - 了解 LightRAG 解决的核心痛点（为什么需要它）。
> - 完成本地开发环境的安装。
> - 编写并运行第一个完整的 LightRAG 程序。

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

# 4. 复制环境变量模板
cp env.example .env
```

## 3. 你的第一个 LightRAG 程序

下面是一个完整可运行的示例。在这个例子中，我们将使用 OpenAI 的模型进行文档解析和查询。

> ⚠️ **关键注意**：在实例化 `LightRAG` 之后，**必须**使用 `await rag.initialize_storages()` 初始化存储后端，否则程序会抛出异常。

```python
import os
import asyncio
from lightrag import LightRAG, QueryParam
from lightrag.llm.openai import gpt_4o_mini_complete, openai_embed
from lightrag.utils import setup_logger

# 启用日志，方便观察实体提取和检索的过程
setup_logger("lightrag", level="INFO")

# 确保配置了 OpenAI API Key
os.environ["OPENAI_API_KEY"] = "sk-你的API密钥"

WORKING_DIR = "./rag_storage"
if not os.path.exists(WORKING_DIR):
    os.mkdir(WORKING_DIR)

async def main():
    rag = None
    try:
        # 1. 初始化 LightRAG 实例
        rag = LightRAG(
            working_dir=WORKING_DIR,
            embedding_func=openai_embed,
            llm_model_func=gpt_4o_mini_complete,
        )
        
        # 2. 强制要求：初始化底层存储
        await rag.initialize_storages()
        
        # 3. 插入文档（在此阶段，LLM 会在后台提取实体和关系）
        document_text = "LightRAG 是香港大学数据科学实验室开源的先进检索系统。它结合了图谱和向量检索。"
        await rag.ainsert(document_text)

        # 4. 执行查询
        # 我们使用 hybrid（混合）模式，它会同时检索局部细节和全局上下文
        query_text = "LightRAG 是由谁开源的？它的核心技术是什么？"
        result = await rag.aquery(
            query_text,
            param=QueryParam(mode="hybrid")
        )
        print("查询结果：\n", result)

    except Exception as e:
        print(f"运行出错: {e}")
    finally:
        # 5. 清理与持久化存储（确保数据落盘）
        if rag:
            await rag.finalize_storages()

if __name__ == "__main__":
    asyncio.run(main())
```

## 4. 下一步学习

恭喜！你已经成功运行了 LightRAG。在真实的生产环境中，我们会有大量的文档和复杂的查询需求。接下来，请阅读 [核心概念：架构与查询模式深度解析](./02-core-architecture.md) 来深入了解其底层机制。

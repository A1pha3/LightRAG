# LightRAG 模型集成：LLM、Embedding 与 Reranker

> 💡 **学习目标**：
> - 掌握如何将开源模型（如 Ollama）或云端大模型（如 Gemini、Azure）接入 LightRAG。
> - 学习如何正确封装和注册自定义 Embedding 函数。
> - 理解 Reranker（重排器）的价值并完成配置。

## 1. 接入本地 LLM（以 Ollama 为例）

LightRAG 需要执行复杂的“实体-关系”提取任务，因此对 LLM 的要求较高：
- **参数量**：建议至少 32B（如 Qwen2.5-32B）。
- **上下文窗口**：至少需要 32KB（建议 64KB）。

默认情况下，Ollama 的模型上下文仅有 8KB。我们**必须**通过 `llm_model_kwargs` 显式扩大上下文窗口。

```python
import asyncio
import numpy as np
from lightrag import LightRAG
from lightrag.utils import wrap_embedding_func_with_attrs
from lightrag.llm.ollama import ollama_model_complete, ollama_embed

# 1. 正确封装 Embedding 函数
# 注意：必须使用 @wrap_embedding_func_with_attrs 装饰器，并声明维度和最大 token
@wrap_embedding_func_with_attrs(
    embedding_dim=768, 
    max_token_size=8192, 
    model_name="nomic-embed-text"
)
async def custom_embedding_func(texts: list[str]) -> np.ndarray:
    # ⚠️ 关键陷阱：必须调用 .func 获取底层的未封装函数
    return await ollama_embed.func(texts, embed_model="nomic-embed-text")

async def main():
    rag = LightRAG(
        working_dir="./rag_storage",
        # 2. 注入生成模型
        llm_model_func=ollama_model_complete,
        llm_model_name="qwen2.5:32b",
        # ⚠️ 关键配置：扩大 Ollama 的上下文窗口至 32KB 以上
        llm_model_kwargs={"options": {"num_ctx": 32768}},
        # 3. 注入 Embedding 模型
        embedding_func=custom_embedding_func,
    )
    
    await rag.initialize_storages()
    
    # 进行文档插入与查询...
    await rag.finalize_storages()

if __name__ == "__main__":
    asyncio.run(main())
```

## 2. 为什么 Embedding 封装如此特殊？

在上面的代码中，我们使用了 `ollama_embed.func`。为什么不能直接传递 `ollama_embed`？

因为 LightRAG 内部使用了严格的类型校验和缓存机制，它需要明确知道 Embedding 向量的维度（`embedding_dim`）。`ollama_embed` 已经被框架底层装饰过一次了，Python 不允许嵌套相同的参数装饰器。因此，我们需要通过 `.func` 拿到原始函数，再重新用我们自己的维度参数进行装饰包装。

> 🛑 **避坑指南**：如果你在中途更换了 Embedding 模型（比如从 768 维换成了 1536 维），你**必须清空工作目录**下的向量数据库文件（但可以保留 `kv_store_llm_response_cache.json` 以节约 LLM API 费用）。

## 3. 引入 Reranker（重排器）提升质量

在 `mix` 模式下，系统会检索出大量来自图谱的边、节点以及纯文本块。将这些内容直接塞给 LLM 可能会导致“幻觉”或超出上下文限制。

**Reranker 的作用**就是作为“裁判”，对这些初步检索到的内容进行二次打分，只保留最相关的 Top-K 内容。

### 配置示例（使用 Jina AI 重排器）：

```python
import os
from lightrag import LightRAG
# 引入 rerank 函数
from lightrag.operate import jina_rerank

# 确保配置了 Jina API Key
os.environ["JINA_API_KEY"] = "你的_JINA_KEY"

rag = LightRAG(
    working_dir="./rag_storage",
    # ... 其他 LLM 和 Embedding 配置 ...
    # 注入重排函数
    rerank_model_func=jina_rerank
)
```

启用 Reranker 后，建议在查询时将 `enable_rerank` 设置为 `True`，并使用 `mix` 模式：

```python
from lightrag import QueryParam

# 假设在某个 async 函数中执行
result = await rag.aquery(
    "详细描述系统的架构",
    param=QueryParam(
        mode="mix",
        enable_rerank=True,
        chunk_top_k=20 # 经过重排后最终保留的文本块数量
    )
)
```

掌握了模型的自定义集成后，对于大规模生产环境，我们需要替换掉本地的 JSON 文件存储。请前往 [专家级存储：图数据库与企业级后端](./04-enterprise-storage.md) 继续学习。

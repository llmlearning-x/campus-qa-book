# 5.1 RAG 核心流程实现

## 从用户提问到 AI 回答

让我们先看完整的代码流程，再逐段解析：

```python
# app/services/rag.py
import logging
from typing import Generator
from openai import OpenAI
from app.core.config import settings
from app.services.embedder import embed_text
from app.services.vector_store import vector_store

logger = logging.getLogger(__name__)
client = OpenAI(api_key=settings.OPENAI_API_KEY, base_url=settings.OPENAI_BASE_URL)

# ============================================================
# 第一步：Prompt 模板
# ============================================================
SYSTEM_PROMPT = """你是一个校园知识问答助手，专门帮助同学解答与学校相关的问题。

## 回答原则

1. **基于资料回答**：只根据下方提供的参考资料作答，不要凭空猜测
2. **诚实面对未知**：如果资料中没有相关信息，明确说「根据现有资料，我无法回答这个问题」
3. **简洁准确**：回答简洁清晰，避免废话
4. **引用来源**：必要时在回答末尾注明信息来源

## 参考资料

{context}
"""


def build_prompt(question: str, retrieved_chunks: list[dict]) -> tuple[str, str]:
    """
    构建 Prompt
    
    Returns:
        (system_prompt, user_message)
    """
    if not retrieved_chunks:
        context = "暂无相关资料。"
    else:
        context_parts = []
        for i, chunk in enumerate(retrieved_chunks, 1):
            context_parts.append(f"【资料{i}】\n{chunk['content']}")
        context = "\n\n".join(context_parts)
    
    system = SYSTEM_PROMPT.format(context=context)
    return system, question


# ============================================================
# 第二步：RAG 检索
# ============================================================
def retrieve(question: str, top_k: int = 5) -> list[dict]:
    """
    RAG 检索：将问题向量化，在知识库中找最相关的片段
    
    Returns:
        [{"chunk_id": int, "content": str, "score": float}, ...]
    """
    query_vector = embed_text(question)
    results = vector_store.search(query_vector, top_k=top_k)
    
    # 过滤掉相关性太低的结果（余弦相似度 < 0.5 认为不相关）
    filtered = [r for r in results if r["score"] >= 0.5]
    
    logger.info(f"检索到 {len(results)} 个片段，过滤后 {len(filtered)} 个（阈值0.5）")
    return filtered


# ============================================================
# 第三步：LLM 生成（流式）
# ============================================================
def generate_stream(question: str) -> Generator[str, None, dict]:
    """
    RAG 问答的流式生成器
    
    Yields:
        str: 每个文字片段（流式输出）
    
    Returns（通过 StopIteration.value）:
        dict: 完整的问答结果，包括来源片段
    """
    # Step 1: 检索相关片段
    retrieved = retrieve(question)
    
    # Step 2: 构建 Prompt
    system_prompt, user_message = build_prompt(question, retrieved)
    
    # Step 3: 调用 LLM（流式模式）
    logger.info(f"调用 LLM，问题：{question[:50]}...")
    
    full_answer = ""
    
    stream = client.chat.completions.create(
        model=settings.CHAT_MODEL,
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_message}
        ],
        stream=True,
        temperature=0.7,
        max_tokens=1500,
    )
    
    for chunk in stream:
        delta = chunk.choices[0].delta
        if delta.content:
            full_answer += delta.content
            yield delta.content  # 实时推送每个字
    
    # 返回完整结果
    return {
        "answer": full_answer,
        "source_chunks": [{"chunk_id": r["chunk_id"], "score": r["score"]} for r in retrieved]
    }
```

---

## 5.2 检索模块：阈值过滤

检索到的结果不一定都相关。余弦相似度分数解读：

| 分数范围 | 含义 |
|---------|------|
| 0.9+ | 高度相关 |
| 0.7~0.9 | 相关 |
| 0.5~0.7 | 有一定相关性 |
| < 0.5 | 可能不相关 |

在 `retrieve()` 函数中，我们过滤掉 `score < 0.5` 的结果。如果过滤后没有任何结果，AI 会诚实地告诉用户没有找到相关信息。

---

## 本节小结

- RAG = 检索（embed + FAISS search）+ 生成（Prompt 拼接 + LLM stream）
- 相似度阈值过滤确保不把无关内容喂给 LLM
- `Generator` 模式支持流式逐字输出

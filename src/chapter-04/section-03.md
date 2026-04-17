# 4.3 向量化处理（Embedding）

## 选择 Embedding 模型

| 模型 | 提供商 | 维度 | 特点 |
|------|--------|------|------|
| text-embedding-3-small | OpenAI | 1536 | 性价比高，支持中文 |
| text-embedding-3-large | OpenAI | 3072 | 精度更高，成本更高 |
| text-embedding-ada-002 | OpenAI | 1536 | 旧版，不推荐新项目 |
| m3e-base | HuggingFace | 768 | 开源中文模型，本地运行 |
| bge-large-zh | HuggingFace | 1024 | 中文效果出色 |

**实训推荐**：`text-embedding-3-small`（支持兼容 OpenAI API 格式的国内服务）

---

## Embedding 服务 `app/services/embedder.py`

```python
import logging
from openai import OpenAI
from app.core.config import settings

logger = logging.getLogger(__name__)

client = OpenAI(
    api_key=settings.OPENAI_API_KEY,
    base_url=settings.OPENAI_BASE_URL,  # 可替换为国内兼容服务
)

def embed_text(text: str) -> list[float]:
    """将单段文本转为向量"""
    response = client.embeddings.create(
        model=settings.EMBEDDING_MODEL,
        input=text,
        encoding_format="float"
    )
    return response.data[0].embedding

def embed_batch(texts: list[str], batch_size: int = 100) -> list[list[float]]:
    """批量向量化（支持大批量，自动分批处理）"""
    all_embeddings = []
    
    for i in range(0, len(texts), batch_size):
        batch = texts[i : i + batch_size]
        logger.info(f"Embedding 批次 {i//batch_size + 1}，共 {len(batch)} 条")
        
        response = client.embeddings.create(
            model=settings.EMBEDDING_MODEL,
            input=batch
        )
        
        # 按原始顺序排序（API 返回顺序可能不一致）
        sorted_data = sorted(response.data, key=lambda x: x.index)
        all_embeddings.extend([item.embedding for item in sorted_data])
    
    return all_embeddings
```

---

## 4.4 向量库构建（FAISS）

### FAISS 基础

FAISS 提供多种索引类型，我们用最简单的两种：

| 索引类型 | 特点 | 适用数据量 |
|---------|------|----------|
| `IndexFlatL2` | 精确搜索，暴力遍历 | < 10万 |
| `IndexIVFFlat` | 近似搜索，速度快 | > 10万 |

对于实训项目，`IndexFlatL2`（欧氏距离）或 `IndexFlatIP`（内积，等效余弦相似度）足够了。

### 向量库管理 `app/services/vector_store.py`

```python
import os
import json
import numpy as np
import faiss
import logging
from app.core.config import settings

logger = logging.getLogger(__name__)

FAISS_INDEX_PATH = os.path.join(settings.UPLOAD_DIR, "faiss.index")
METADATA_PATH    = os.path.join(settings.UPLOAD_DIR, "faiss_meta.json")

class VectorStore:
    """FAISS 向量库管理器（单例模式）"""
    
    def __init__(self):
        self.dimension = 1536  # text-embedding-3-small 的向量维度
        self.index = None
        self.metadata = []  # 存储每个向量对应的元数据（chunk_id, doc_id, content）
        self._load_or_create()
    
    def _load_or_create(self):
        """加载已有索引或创建新索引"""
        os.makedirs(settings.UPLOAD_DIR, exist_ok=True)
        
        if os.path.exists(FAISS_INDEX_PATH) and os.path.exists(METADATA_PATH):
            self.index = faiss.read_index(FAISS_INDEX_PATH)
            with open(METADATA_PATH, "r", encoding="utf-8") as f:
                self.metadata = json.load(f)
            logger.info(f"加载 FAISS 索引：{self.index.ntotal} 个向量")
        else:
            self.index = faiss.IndexFlatIP(self.dimension)  # 内积（余弦相似度）
            self.metadata = []
            logger.info("创建新的 FAISS 索引")
    
    def add_vectors(
        self, 
        vectors: list[list[float]], 
        chunk_ids: list[int],
        contents: list[str]
    ) -> list[int]:
        """
        添加向量到索引
        
        Returns:
            每个向量在 FAISS 中的 ID 列表
        """
        start_id = self.index.ntotal
        
        # 归一化（余弦相似度需要）
        arr = np.array(vectors, dtype=np.float32)
        faiss.normalize_L2(arr)
        
        self.index.add(arr)
        
        # 记录元数据
        vector_ids = []
        for i, (chunk_id, content) in enumerate(zip(chunk_ids, contents)):
            vector_id = start_id + i
            self.metadata.append({
                "vector_id": vector_id,
                "chunk_id": chunk_id,
                "content": content
            })
            vector_ids.append(vector_id)
        
        self._save()
        return vector_ids
    
    def search(self, query_vector: list[float], top_k: int = 5) -> list[dict]:
        """
        检索最相似的 top_k 个片段
        
        Returns:
            [{"content": str, "chunk_id": int, "score": float}, ...]
        """
        if self.index.ntotal == 0:
            return []
        
        arr = np.array([query_vector], dtype=np.float32)
        faiss.normalize_L2(arr)
        
        scores, indices = self.index.search(arr, min(top_k, self.index.ntotal))
        
        results = []
        for score, idx in zip(scores[0], indices[0]):
            if idx < 0:  # FAISS 用 -1 表示无效结果
                continue
            meta = self.metadata[idx]
            results.append({
                "chunk_id": meta["chunk_id"],
                "content": meta["content"],
                "score": float(score)
            })
        
        return results
    
    def remove_by_chunk_ids(self, chunk_ids: set[int]):
        """删除指定 chunk_id 的向量（重建索引）"""
        # FAISS 不支持直接删除，需要重建
        keep = [m for m in self.metadata if m["chunk_id"] not in chunk_ids]
        
        if not keep:
            self.index = faiss.IndexFlatIP(self.dimension)
            self.metadata = []
        else:
            vectors_to_keep = np.zeros((len(keep), self.dimension), dtype=np.float32)
            new_metadata = []
            
            for i, m in enumerate(keep):
                idx = m["vector_id"]
                # 从原索引重建（注意：仅适合小规模数据）
                vec = self.index.reconstruct(idx)
                vectors_to_keep[i] = vec
                new_metadata.append({**m, "vector_id": i})
            
            self.index = faiss.IndexFlatIP(self.dimension)
            faiss.normalize_L2(vectors_to_keep)
            self.index.add(vectors_to_keep)
            self.metadata = new_metadata
        
        self._save()
        logger.info(f"删除向量完成，剩余 {self.index.ntotal} 个")
    
    def _save(self):
        """持久化索引到磁盘"""
        faiss.write_index(self.index, FAISS_INDEX_PATH)
        with open(METADATA_PATH, "w", encoding="utf-8") as f:
            json.dump(self.metadata, f, ensure_ascii=False, indent=2)

# 全局单例
vector_store = VectorStore()
```

---

## 本节小结

- `text-embedding-3-small` 是入门首选，支持中文，按 API 调用计费
- 批量 Embedding 比逐条调用效率高得多
- FAISS `IndexFlatIP` 使用归一化向量实现余弦相似度搜索
- 向量库需要与 MySQL 元数据同步维护（chunk_id 是桥梁）

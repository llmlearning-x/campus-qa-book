# 4.4 向量库构建（FAISS）

_本节内容已并入 [4.3 向量化处理（Embedding）](section-03.md) 中的「向量库管理」部分。_

请参阅 4.3 节中的 `VectorStore` 完整实现，涵盖：
- `IndexFlatIP` 索引创建与加载
- `add_vectors()` 批量写入向量
- `search()` TopK 语义检索
- `remove_by_chunk_ids()` 向量删除与索引重建
- `_save()` 持久化到磁盘

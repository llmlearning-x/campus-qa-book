# 5.2 检索模块开发（TopK）

_本节内容已整合在 [5.1 RAG 核心流程实现](section-01.md) 中的 `retrieve()` 函数部分。_

**要点回顾**：
- `embed_text(question)` 将问题向量化
- `vector_store.search(query_vector, top_k=5)` 在 FAISS 中找最相似的 5 个片段
- 相似度阈值过滤：`score >= 0.5` 才保留

---

# 5.4 大模型 API 调用（LLM 接入）

## 国内替代方案

如果无法访问 OpenAI，可使用以下兼容 OpenAI API 格式的国内服务：

| 服务商 | 模型 | base_url | 特点 |
|--------|------|---------|------|
| 阿里云百炼 | qwen-turbo / qwen-plus | `https://dashscope.aliyuncs.com/compatible-mode/v1` | 通义千问，国内访问 |
| 字节跳动火山引擎 | doubao | `https://ark.cn-beijing.volces.com/api/v3` | 豆包，快 |
| 硅基流动 | deepseek-v3 | `https://api.siliconflow.cn/v1` | 便宜，DeepSeek |

**切换方式**（只需改 `.env`）：
```ini
OPENAI_API_KEY=your-api-key
OPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
CHAT_MODEL=qwen-turbo
EMBEDDING_MODEL=text-embedding-v3
```

代码无需修改，因为我们已经通过 `settings.OPENAI_BASE_URL` 参数化了 API 地址。

---

# 5.6 问答记录模块 CRUD

_问答记录的保存逻辑已在 [5.5 问答接口开发](section-05.md) 的 `event_generator()` 中实现。_

**历史记录查询接口**已在 `app/api/chat.py` 中的 `GET /api/chat/history` 实现。

---

# 5.8 问答效果测试与优化

## 测试方法

创建测试文档（`test_campus.txt`）：
```
中央食堂营业时间：早餐 7:00-9:00，午餐 11:00-13:30，晚餐 17:00-19:30。
```

上传后，依次提问：
1. "食堂几点开门？" → 应回答早餐7:00
2. "食堂下午几点开始营业？" → 应回答晚餐17:00
3. "图书馆什么时候关门？" → 应回答无法找到相关信息

## 常见问题与调优

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 回答不准确 | chunk_size 太大 | 减小到 300 字 |
| 检索不到相关内容 | 相似度阈值太高 | 降低阈值到 0.4 |
| 回答编造内容 | Prompt 约束不够 | 加强「只基于资料」指令 |
| 响应太慢 | 模型选择 | 换更快的模型（如 qwen-turbo） |

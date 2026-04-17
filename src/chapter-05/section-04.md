# 5.4 大模型 API 调用

_本节内容已整合在 [5.2 检索模块开发](section-02.md) 中的「国内替代方案」部分，以及 [5.1 RAG核心](section-01.md) 中的 LLM 调用实现。_

关键配置参数总结：
- `OPENAI_BASE_URL`：兼容 OpenAI 格式的 API 地址，支持国内服务商
- `CHAT_MODEL`：生成模型（如 `gpt-4o-mini`、`qwen-turbo`）
- `temperature=0.7`：适合问答场景
- `max_tokens=1500`：限制回答长度，控制成本
- `stream=True`：开启流式响应

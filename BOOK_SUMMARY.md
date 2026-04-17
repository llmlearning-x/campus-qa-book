# 全书压缩摘要 — BOOK_SUMMARY.md

> 每章≤500字摘要，用于逐章写作时加载，避免上下文窗口溢出。

---

## 第1章：项目启动与技术准备

本章奠定认知基础。介绍软件开发的完整生命周期（需求→设计→开发→测试→部署），以及敏捷开发理念。重点讲解三个 AI 核心概念：LLM（大语言模型，理解和生成文本）、Embedding（将文本转为高维向量，使语义相近的文本在数学上也相近）、RAG（先检索相关文档片段，再让 LLM 基于这些片段回答，解决 LLM 不了解私有数据和上下文长度限制的问题）。给出系统架构：Vue3 前端 + FastAPI 后端 + MySQL 关系数据库 + FAISS 向量库 + LLM API。完成 Python/Node.js/MySQL 开发环境搭建。

---

## 第2章：工程基础搭建

搭建前后端工程骨架。后端 FastAPI 项目初始化，分层结构（api/models/schemas/services/core），用 pydantic-settings 管理配置，环境变量存 .env 不提交 Git。前端 Vue3+Vite 初始化，Axios 拦截器统一携带 Token 和处理错误，Vite proxy 解决开发环境跨域。数据库设计 4 张核心表：users（用户）、documents（文档元数据）、document_chunks（切分后片段）、qa_records（问答历史）。GitLab 分支策略：main（稳定）→ dev（集成）→ feature/xxx（功能分支）。

---

## 第3章：用户系统与权限控制

实现完整的认证与授权体系。密码用 bcrypt 哈希存储。JWT Token 认证：无状态、支持水平扩展，Payload 包含 user_id 和 role，有过期时间。FastAPI 依赖注入 `Depends(get_current_user)` 在接口层透明完成认证，`require_admin` 实现管理员权限检查。用户 CRUD 使用软删除（is_active=0）。前端 Pinia 存储登录状态，Vue Router 路由守卫自动拦截未登录和越权访问。Element Plus 实现登录页、注册页、用户管理表格页。

---

## 第4章：知识库与向量化处理

实现 RAG 的「记忆建立」Pipeline。文档上传用 FastAPI BackgroundTasks 异步处理，不阻塞 HTTP 响应。文本切分：LangChain RecursiveCharacterTextSplitter，默认 500 字/片段，50 字重叠，优先在段落/句子边界切割。向量化：调用 OpenAI text-embedding-3-small API 批量处理，支持国内兼容 API 替换。向量库：FAISS IndexFlatIP（内积实现余弦相似度），持久化到磁盘，支持增删查。删除文档时同步清理 FAISS 向量（需重建索引，小规模可接受）。前端 Knowledge.vue 实现文件上传+状态轮询+列表管理。

---

## 第5章：RAG 核心与问答接口

串联所有组件实现完整 RAG 问答。核心流程：用户问题 → embed_text() → FAISS TopK 检索（默认 Top5，过滤 score<0.5）→ 构建 Prompt（系统指令+检索片段+问题）→ LLM 流式生成。Prompt 设计防幻觉：明确约束「只基于参考资料回答」「找不到信息诚实说」。LLM API 通过配置 OPENAI_BASE_URL 支持 OpenAI/阿里云百炼/硅基流动等服务商。流式响应用 FastAPI StreamingResponse + SSE 格式（data: {...}\n\n），前端用 fetch + ReadableStream 接收。问答记录在流式完成后异步保存到 MySQL。聊天界面：气泡布局+打字机效果+Markdown 渲染。

---

## 第6章：测试、发布与项目总结

系统联调测试覆盖：认证流程、知识库管理、问答流程、权限控制的正常和异常路径。前端生产构建（npm run build），Nginx 托管静态文件并反向代理 API（注意 SSE 必须关闭 proxy_buffering）。后端用 uvicorn + supervisor 进程守护部署。总结技术债务（FAISS 删除性能、缺少单测等）和后续优化方向（多模态、混合检索、Reranker、容器化）。人员评估表记录每位成员的角色贡献和技术掌握度。

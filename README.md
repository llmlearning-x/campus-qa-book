# 从零构建校园问答助手：RAG全栈实战开发指南

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

> 面向在校大学生和实训学员的 RAG 全栈实战教程，6天从零搭建校园知识问答系统

## 📖 关于本书

本书来自一个真实的实训项目：6天时间，带领大学生从第一行代码开始，完整交付一个校园知识问答助手——它能上传文档、理解文档内容，并用自然语言回答用户的问题。

## 🗂 书籍结构

| 章节 | 主题 | 核心内容 |
|------|------|---------|
| 第1章 | 项目启动与技术准备 | LLM/Embedding/RAG原理、架构设计、环境搭建 |
| 第2章 | 工程基础搭建 | FastAPI+Vue3脚手架、数据库设计、GitLab |
| 第3章 | 用户系统与权限控制 | JWT认证、用户CRUD、路由守卫 |
| 第4章 | 知识库与向量化处理 | 文档切分、Embedding、FAISS向量库 |
| 第5章 | RAG核心与问答接口 | RAG流程、Prompt工程、SSE流式响应 |
| 第6章 | 测试、发布与项目总结 | 联调测试、部署、项目复盘 |

## 🛠 技术栈

- **前端**：Vue3 + Vite + Element Plus + Pinia
- **后端**：Python + FastAPI + SQLAlchemy
- **数据库**：MySQL 8.0
- **向量库**：FAISS
- **LLM**：兼容 OpenAI API 格式（支持 GPT-4o / 通义千问 / DeepSeek 等）

## 🚀 快速开始

```bash
# 1. 克隆仓库
git clone https://github.com/llmlearning-x/campus-qa-book.git
cd campus-qa-book

# 2. 阅读书籍（需要安装 mdBook）
mdbook serve
# 访问 http://localhost:3000
```

## 📄 License

MIT

---

_版本 v1.0 | 2026年4月_

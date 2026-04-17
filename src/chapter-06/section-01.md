# 6.1 完善其他模块

## 常见遗漏项检查清单

在联调之前，先自查这些常见遗漏点：

### 后端
- [ ] 所有接口都有正确的权限控制（不该公开的接口加了 `Depends(get_current_user)`）
- [ ] 文件上传目录 `uploads/` 已创建，并加入 `.gitignore`
- [ ] FAISS 索引文件已加入 `.gitignore`
- [ ] `.env` 文件有完整的配置项，新团队成员能按 `.env.example` 快速配置
- [ ] 数据库连接超时有重试机制

### 前端
- [ ] 所有 API 请求错误都有用户友好的提示（不能让用户看到 `[object Object]`）
- [ ] Loading 状态：上传文档时、发送消息时有加载指示
- [ ] 空状态处理：文档列表为空、历史记录为空时有提示文字
- [ ] 移动端基本可用（最小宽度 375px 不会溢出）

---

## 创建 .env.example

```ini
# .env.example（提交到 Git，让新成员知道需要哪些配置）
DATABASE_URL=mysql+pymysql://用户名:密码@localhost/campus_qa
SECRET_KEY=修改为随机字符串
OPENAI_API_KEY=你的API密钥
OPENAI_BASE_URL=https://api.openai.com/v1
EMBEDDING_MODEL=text-embedding-3-small
CHAT_MODEL=gpt-4o-mini
UPLOAD_DIR=./uploads
```

---

## 本节小结

- 发布前的自查清单帮助发现遗漏项
- `.env.example` 让团队协作更顺畅
- 空状态和错误提示是 UI 完整性的重要部分

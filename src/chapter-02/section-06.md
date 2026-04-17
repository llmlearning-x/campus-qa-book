# 2.6 接口设计（Swagger）

## 接口文档是前后端协作的契约

在团队开发中，前端和后端同时开工是常态。前端不需要等后端写完接口才开始，只需要在接口文档里约定好：

- 这个接口的 URL 是什么？
- 需要传哪些参数？
- 返回的数据格式是什么？

**FastAPI 的杀手锏**：只要你写了代码，Swagger 文档自动生成，零额外工作量。

---

## RESTful 设计规范

REST（Representational State Transfer）是一种 API 设计风格，核心规则：

| 操作 | HTTP 方法 | URL 示例 | 说明 |
|------|----------|---------|------|
| 获取列表 | GET | `/api/users` | 获取用户列表 |
| 获取单个 | GET | `/api/users/1` | 获取 ID=1 的用户 |
| 创建 | POST | `/api/users` | 创建新用户 |
| 更新 | PUT/PATCH | `/api/users/1` | 更新 ID=1 的用户 |
| 删除 | DELETE | `/api/users/1` | 删除 ID=1 的用户 |

**URL 命名规则**：
- 用名词，不用动词：`/api/users`（✅）而不是 `/api/getUsers`（❌）
- 用复数：`/api/users` 而不是 `/api/user`
- 用小写 + 连字符：`/api/qa-records`

---

## 统一响应格式

前后端约定一个统一的响应结构，前端处理起来更方便：

```python
# app/schemas/response.py
from typing import Generic, TypeVar, Optional
from pydantic import BaseModel

T = TypeVar('T')

class ResponseModel(BaseModel, Generic[T]):
    code: int = 200
    message: str = "success"
    data: Optional[T] = None

class PageData(BaseModel, Generic[T]):
    total: int
    items: list[T]
    page: int
    page_size: int
```

**使用示例**：

```python
# 成功响应
return ResponseModel(data={"user_id": 1, "username": "张三"})
# → {"code": 200, "message": "success", "data": {...}}

# 分页响应
return ResponseModel(data=PageData(total=100, items=[...], page=1, page_size=10))

# 错误响应（通过 HTTPException）
raise HTTPException(status_code=400, detail="用户名已存在")
```

---

## 核心接口清单

以下是我们系统的完整接口列表，按模块分组：

### 用户模块 `/api/users`

| 方法 | URL | 说明 | 权限 |
|------|-----|------|------|
| POST | `/register` | 用户注册 | 无需登录 |
| POST | `/login` | 用户登录，返回 Token | 无需登录 |
| GET | `/me` | 获取当前用户信息 | 需要登录 |
| GET | `/` | 获取用户列表（分页） | 管理员 |
| GET | `/{user_id}` | 获取指定用户 | 管理员 |
| PUT | `/{user_id}` | 修改用户信息 | 管理员 |
| DELETE | `/{user_id}` | 禁用/删除用户 | 管理员 |

### 知识库模块 `/api/knowledge`

| 方法 | URL | 说明 | 权限 |
|------|-----|------|------|
| POST | `/documents` | 上传文档 | 管理员 |
| GET | `/documents` | 文档列表（分页） | 需要登录 |
| GET | `/documents/{doc_id}` | 文档详情 | 需要登录 |
| DELETE | `/documents/{doc_id}` | 删除文档 | 管理员 |
| POST | `/documents/{doc_id}/reprocess` | 重新处理文档 | 管理员 |

### 问答模块 `/api`

| 方法 | URL | 说明 | 权限 |
|------|-----|------|------|
| POST | `/chat` | 发起问答（流式） | 需要登录 |
| GET | `/chat/history` | 历史记录 | 需要登录 |

---

## 查看 Swagger 文档

启动后端后，访问：
- `http://localhost:8000/docs` — Swagger UI（交互式，可直接测试接口）
- `http://localhost:8000/redoc` — ReDoc（更美观的文档格式）

在 Swagger UI 里，你可以：
1. 点击接口展开详情
2. 点击 "Try it out" 填写参数
3. 点击 "Execute" 直接发送请求
4. 查看响应结果

---

## 第2章小结

🎉 Day 2 完成！

**今天你做了什么**：
- ✅ MySQL 数据库搭建，4 张核心表创建完成
- ✅ FastAPI 后端项目初始化，Swagger 文档可访问
- ✅ Vue3 前端脚手架搭建，Axios 请求封装完成
- ✅ 前后端联调成功
- ✅ 代码推送到 GitLab，分支结构建立

**第2天的产出物**：
- [ ] GitLab 项目仓库（前端 + 后端）
- [ ] 初始化代码已推送到 main 分支
- [ ] 联调成功截图
- [ ] 完整的建表 SQL 脚本（`init.sql`）

**明天（Day 3）** 开始写第一个真正的业务功能：用户注册、登录、JWT 认证、权限控制。

---

_第2章完_

# 2.2 前后端脚手架搭建

## 后端：FastAPI 项目初始化

### 项目结构

创建以下目录结构：

```bash
mkdir -p app/{api,models,schemas,services,core}
touch app/__init__.py app/main.py
touch app/api/__init__.py app/models/__init__.py
touch app/schemas/__init__.py app/services/__init__.py
touch app/core/__init__.py app/core/config.py app/core/database.py
```

### 配置管理 `app/core/config.py`

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # 数据库
    DATABASE_URL: str = "mysql+pymysql://campus:campus123456@localhost/campus_qa"
    
    # JWT 认证
    SECRET_KEY: str = "change-this-to-a-random-secret-key"
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30 * 24 * 60  # 30天
    
    # OpenAI
    OPENAI_API_KEY: str = ""
    OPENAI_BASE_URL: str = "https://api.openai.com/v1"
    EMBEDDING_MODEL: str = "text-embedding-3-small"
    CHAT_MODEL: str = "gpt-4o-mini"
    
    # 文件上传
    UPLOAD_DIR: str = "./uploads"
    MAX_FILE_SIZE: int = 50 * 1024 * 1024  # 50MB
    
    class Config:
        env_file = ".env"  # 从 .env 文件加载配置

settings = Settings()
```

### 入口文件 `app/main.py`

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.api import user, knowledge, chat
from app.core.database import engine, Base

# 创建数据库表（开发阶段使用，生产环境用 Alembic 迁移）
Base.metadata.create_all(bind=engine)

app = FastAPI(
    title="校园问答助手 API",
    description="基于 RAG 的校园知识问答系统",
    version="1.0.0"
)

# 配置 CORS（允许前端跨域访问）
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173", "http://127.0.0.1:5173"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 注册路由
app.include_router(user.router, prefix="/api/users", tags=["用户管理"])
app.include_router(knowledge.router, prefix="/api/knowledge", tags=["知识库"])
app.include_router(chat.router, prefix="/api", tags=["问答"])

@app.get("/")
def root():
    return {"message": "校园问答助手 API 运行中", "docs": "/docs"}

@app.get("/health")
def health():
    return {"status": "ok"}
```

### 启动后端

```bash
# 激活虚拟环境
source venv/bin/activate

# 启动开发服务器（--reload 代码修改后自动重启）
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

访问 `http://localhost:8000/docs` 查看 Swagger 文档，如果能看到接口列表，说明后端启动成功！

---

## 前端：Vue3 + Vite 项目初始化

### 项目结构规范

```
campus-qa-frontend/
├── src/
│   ├── api/           # API 请求封装
│   │   ├── request.js # axios 实例配置
│   │   ├── user.js    # 用户相关 API
│   │   └── chat.js    # 问答相关 API
│   ├── stores/        # Pinia 状态管理
│   │   └── user.js    # 用户状态（登录信息）
│   ├── router/        # 路由配置
│   │   └── index.js
│   ├── views/         # 页面组件
│   │   ├── Login.vue
│   │   ├── Chat.vue
│   │   └── Knowledge.vue
│   ├── components/    # 可复用组件
│   ├── App.vue
│   └── main.js
├── index.html
├── vite.config.js
└── package.json
```

### Axios 请求封装 `src/api/request.js`

```javascript
import axios from 'axios'
import { ElMessage } from 'element-plus'

// 创建 axios 实例
const request = axios.create({
  baseURL: 'http://localhost:8000',  // 后端地址
  timeout: 30000,                    // 30秒超时
})

// 请求拦截器：自动携带 Token
request.interceptors.request.use(config => {
  const token = localStorage.getItem('token')
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

// 响应拦截器：统一错误处理
request.interceptors.response.use(
  response => response.data,
  error => {
    const status = error.response?.status
    if (status === 401) {
      localStorage.removeItem('token')
      window.location.href = '/login'
      ElMessage.error('登录已过期，请重新登录')
    } else if (status === 403) {
      ElMessage.error('没有权限')
    } else {
      ElMessage.error(error.response?.data?.detail || '请求失败')
    }
    return Promise.reject(error)
  }
)

export default request
```

### Vite 配置（解决开发环境跨域）`vite.config.js`

```javascript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { fileURLToPath, URL } from 'node:url'

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url))
    },
  },
  server: {
    port: 5173,
    proxy: {
      // 开发环境代理，避免跨域问题
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true,
      }
    }
  }
})
```

---

## 前后端联调验证

在后端 `app/api/user.py` 创建一个测试接口：

```python
from fastapi import APIRouter

router = APIRouter()

@router.get("/ping")
def ping():
    return {"message": "后端连接成功！", "status": "ok"}
```

在前端 `src/views/Home.vue` 调用它：

```vue
<template>
  <div>
    <h1>校园问答助手</h1>
    <p>后端状态：{{ status }}</p>
    <button @click="testConnection">测试连接</button>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import request from '@/api/request'

const status = ref('未连接')

async function testConnection() {
  try {
    const data = await request.get('/api/users/ping')
    status.value = data.message
  } catch (e) {
    status.value = '连接失败：' + e.message
  }
}
</script>
```

如果点击按钮后显示「后端连接成功！」，说明前后端联调通过 🎉

---

## 本节小结

- FastAPI 项目结构清晰：api / models / schemas / services / core 分层
- 配置管理用 pydantic-settings，环境变量与代码分离
- CORS 配置允许前端跨域访问后端
- Axios 拦截器统一处理 Token 和错误，减少重复代码

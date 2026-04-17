# 2.4 Demo 程序运行测试

## 目标：让前后端真正「说上话」

在功能开发开始之前，我们先验证整个技术链路是通的：

```
前端 Vue3 → HTTP 请求 → 后端 FastAPI → 数据库 MySQL → 返回数据 → 前端展示
```

---

## 后端 Hello World

创建最简单的接口 `app/api/demo.py`：

```python
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session
from sqlalchemy import text
from app.core.database import get_db

router = APIRouter()

@router.get("/hello")
def hello():
    """最简单的测试接口"""
    return {
        "message": "Hello from Campus QA Backend!",
        "version": "1.0.0"
    }

@router.get("/db-check")
def db_check(db: Session = Depends(get_db)):
    """测试数据库连接"""
    try:
        result = db.execute(text("SELECT 1")).scalar()
        return {"database": "connected", "result": result}
    except Exception as e:
        return {"database": "error", "detail": str(e)}
```

在 `main.py` 里注册这个路由：
```python
from app.api import demo
app.include_router(demo.router, prefix="/api/demo", tags=["演示"])
```

访问 `http://localhost:8000/api/demo/hello` 和 `/api/demo/db-check`，验证两个接口都返回正常结果。

---

## 前端联调页面

创建 `src/views/Demo.vue`：

```vue
<template>
  <div style="padding: 40px; max-width: 600px; margin: 0 auto;">
    <h2>系统状态检查</h2>
    
    <el-card style="margin-top: 20px;">
      <template #header>API 连接测试</template>
      <el-space direction="vertical" style="width: 100%">
        <div>
          <el-button type="primary" @click="testHello" :loading="loading.hello">
            测试 Hello 接口
          </el-button>
          <el-tag v-if="results.hello" :type="results.hello.ok ? 'success' : 'danger'" style="margin-left: 12px;">
            {{ results.hello.message }}
          </el-tag>
        </div>
        <div>
          <el-button type="primary" @click="testDB" :loading="loading.db">
            测试数据库连接
          </el-button>
          <el-tag v-if="results.db" :type="results.db.database === 'connected' ? 'success' : 'danger'" style="margin-left: 12px;">
            {{ results.db.database === 'connected' ? '数据库连接正常' : '数据库连接失败' }}
          </el-tag>
        </div>
      </el-space>
    </el-card>
  </div>
</template>

<script setup>
import { ref, reactive } from 'vue'
import request from '@/api/request'

const loading = reactive({ hello: false, db: false })
const results = reactive({ hello: null, db: null })

async function testHello() {
  loading.hello = true
  try {
    const data = await request.get('/api/demo/hello')
    results.hello = { ok: true, message: data.message }
  } catch (e) {
    results.hello = { ok: false, message: '请求失败' }
  } finally {
    loading.hello = false
  }
}

async function testDB() {
  loading.db = true
  try {
    const data = await request.get('/api/demo/db-check')
    results.db = data
  } catch (e) {
    results.db = { database: 'error' }
  } finally {
    loading.db = false
  }
}
</script>
```

---

## 前后端联调成功标志

当你在浏览器里点击按钮，能看到：
- ✅ "Hello from Campus QA Backend!" 绿色标签
- ✅ "数据库连接正常" 绿色标签

截图保存，作为 Day 2 的联调成功凭证。

---

## 本节小结

- Demo 接口验证：API 层通了、数据库通了
- 前端用 Element Plus 展示状态，视觉清晰
- 保存联调成功截图，作为进度凭证

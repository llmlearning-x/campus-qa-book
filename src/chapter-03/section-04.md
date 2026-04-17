# 3.4 登录认证：JWT + Token

## 为什么不用 Session？

传统的 Web 应用使用 **Session** 记录登录状态：用户登录后，服务器生成一个 Session ID 存在内存里，浏览器通过 Cookie 携带这个 ID。

这种方式有个问题：**水平扩展困难**。如果你的系统部署了 3 台服务器，用户的 Session 只在其中一台上，下次请求可能被路由到另一台，Session 就找不到了。

**JWT（JSON Web Token）** 解决了这个问题：Token 本身包含了用户信息，服务器不需要存储任何状态，每台服务器都能独立验证。

---

## JWT 原理

一个 JWT Token 由三部分组成，用 `.` 分隔：

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9    ← Header（算法类型）
.eyJ1c2VyX2lkIjoxLCJleHAiOjE3MDB9        ← Payload（用户数据）
.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c  ← Signature（签名）
```

**Payload 里存什么**：
```json
{
  "user_id": 1,
  "username": "张三",
  "role": "user",
  "exp": 1700000000   ← 过期时间（Unix 时间戳）
}
```

**安全性保证**：Signature 是用服务器的 `SECRET_KEY` 对 Header + Payload 进行 HMAC 签名。攻击者无法伪造 Token，因为他不知道 SECRET_KEY。

---

## 后端实现

### 安装依赖
```bash
pip install python-jose[cryptography] passlib[bcrypt]
```

### 认证工具 `app/services/auth.py`
```python
from datetime import datetime, timedelta
from typing import Optional
from jose import JWTError, jwt
from passlib.context import CryptContext
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.orm import Session
from app.core.config import settings
from app.core.database import get_db
from app.models.models import User

# 密码哈希工具（bcrypt 算法）
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# OAuth2 scheme（从 Authorization: Bearer <token> 头取 Token）
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/users/login")

def hash_password(password: str) -> str:
    """将明文密码转为 bcrypt 哈希"""
    return pwd_context.hash(password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """验证密码"""
    return pwd_context.verify(plain_password, hashed_password)

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None):
    """生成 JWT Token"""
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(
        minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES
    ))
    to_encode["exp"] = expire
    return jwt.encode(to_encode, settings.SECRET_KEY, algorithm=settings.ALGORITHM)

def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: Session = Depends(get_db)
) -> User:
    """从 Token 获取当前登录用户（FastAPI 依赖注入）"""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="无效的认证凭证",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, settings.SECRET_KEY, algorithms=[settings.ALGORITHM])
        user_id: int = payload.get("user_id")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception
    
    user = db.query(User).filter(User.id == user_id, User.is_active == 1).first()
    if user is None:
        raise credentials_exception
    return user

def require_admin(current_user: User = Depends(get_current_user)) -> User:
    """要求管理员权限"""
    if current_user.role != "admin":
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="需要管理员权限"
        )
    return current_user
```

### 用户接口 `app/api/user.py`
```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from app.core.database import get_db
from app.models.models import User
from app.services.auth import (
    hash_password, verify_password, create_access_token,
    get_current_user, require_admin
)
from pydantic import BaseModel

router = APIRouter()

class RegisterRequest(BaseModel):
    username: str
    password: str
    email: str = None

class LoginRequest(BaseModel):
    username: str
    password: str

@router.post("/register")
def register(req: RegisterRequest, db: Session = Depends(get_db)):
    """用户注册"""
    if db.query(User).filter(User.username == req.username).first():
        raise HTTPException(status_code=400, detail="用户名已存在")
    
    user = User(
        username=req.username,
        password=hash_password(req.password),
        email=req.email
    )
    db.add(user)
    db.commit()
    db.refresh(user)
    return {"message": "注册成功", "user_id": user.id}

@router.post("/login")
def login(req: LoginRequest, db: Session = Depends(get_db)):
    """用户登录"""
    user = db.query(User).filter(
        User.username == req.username, User.is_active == 1
    ).first()
    
    if not user or not verify_password(req.password, user.password):
        raise HTTPException(status_code=401, detail="用户名或密码错误")
    
    token = create_access_token({"user_id": user.id, "role": user.role})
    return {
        "access_token": token,
        "token_type": "bearer",
        "user": {"id": user.id, "username": user.username, "role": user.role}
    }

@router.get("/me")
def get_me(current_user: User = Depends(get_current_user)):
    """获取当前登录用户信息"""
    return {
        "id": current_user.id,
        "username": current_user.username,
        "email": current_user.email,
        "role": current_user.role
    }
```

---

## 前端登录页面

`src/views/Login.vue`：
```vue
<template>
  <div class="login-page">
    <el-card class="login-card">
      <h2>校园问答助手</h2>
      <el-form :model="form" :rules="rules" ref="formRef">
        <el-form-item prop="username">
          <el-input v-model="form.username" placeholder="用户名" prefix-icon="User" />
        </el-form-item>
        <el-form-item prop="password">
          <el-input v-model="form.password" type="password" placeholder="密码"
            prefix-icon="Lock" show-password @keyup.enter="handleLogin" />
        </el-form-item>
        <el-button type="primary" :loading="loading" style="width:100%" @click="handleLogin">
          登录
        </el-button>
      </el-form>
    </el-card>
  </div>
</template>

<script setup>
import { ref, reactive } from 'vue'
import { useRouter } from 'vue-router'
import { ElMessage } from 'element-plus'
import { useUserStore } from '@/stores/user'

const router = useRouter()
const userStore = useUserStore()
const formRef = ref()
const loading = ref(false)

const form = reactive({ username: '', password: '' })
const rules = {
  username: [{ required: true, message: '请输入用户名' }],
  password: [{ required: true, message: '请输入密码' }]
}

async function handleLogin() {
  await formRef.value.validate()
  loading.value = true
  try {
    await userStore.login(form.username, form.password)
    ElMessage.success('登录成功')
    router.push('/chat')
  } catch (e) {
    // 错误已由 axios 拦截器处理
  } finally {
    loading.value = false
  }
}
</script>

<style scoped>
.login-page {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #f0f2f5;
}
.login-card {
  width: 380px;
  padding: 20px;
  text-align: center;
}
</style>
```

---

## 本节小结

- JWT 是无状态的认证机制，天然支持水平扩展
- `passlib` + `bcrypt` 安全哈希密码，绝不存明文
- `Depends(get_current_user)` 是 FastAPI 权限控制的核心模式
- 前端把 Token 存 `localStorage`，每次请求由 axios 拦截器自动携带

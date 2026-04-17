# 3.2 后端基础接口开发

## FastAPI 路由与依赖注入

FastAPI 的核心优势是基于 Python 类型注解自动完成数据验证和文档生成，并且提供了强大的**依赖注入（Dependency Injection）**系统。

```python
# 依赖注入的基本用法
from fastapi import Depends

def get_db():    # 一个依赖项
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/users")
def list_users(db = Depends(get_db)):   # 注入数据库会话
    return db.query(User).all()
```

`Depends()` 告诉 FastAPI：执行这个接口之前，先执行 `get_db()`，把结果注入到 `db` 参数里。FastAPI 会自动管理依赖的生命周期。

---

## 统一异常处理

```python
# app/main.py 中添加
from fastapi import Request
from fastapi.responses import JSONResponse

@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    return JSONResponse(
        status_code=exc.status_code,
        content={"code": exc.status_code, "message": exc.detail, "data": None}
    )

@app.exception_handler(Exception)
async def general_exception_handler(request: Request, exc: Exception):
    return JSONResponse(
        status_code=500,
        content={"code": 500, "message": "服务器内部错误", "data": None}
    )
```

---

## 日志配置

```python
# app/core/logger.py
import logging

def setup_logging():
    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S"
    )

logger = logging.getLogger("campus-qa")
```

在接口里使用：
```python
from app.core.logger import logger

@router.post("/register")
def register(req: RegisterRequest, db: Session = Depends(get_db)):
    logger.info(f"新用户注册：{req.username}")
    # ...
```

---

## 本节小结

- `Depends()` 是 FastAPI 依赖注入的核心，用于共享数据库连接、认证逻辑等
- 统一异常处理让所有错误返回格式一致，前端处理更方便
- 日志记录关键操作，便于问题排查

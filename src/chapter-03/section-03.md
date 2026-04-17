# 3.3 用户模块 CRUD

## 完整的用户管理接口

用户模块是整个系统的基础。我们需要实现：
- 创建用户（注册）
- 读取用户（列表、详情）
- 更新用户（修改信息）
- 删除用户（禁用/软删除）

---

## Pydantic Schema 定义

Pydantic 是 FastAPI 的数据验证库。定义请求/响应的数据结构：

```python
# app/schemas/user.py
from pydantic import BaseModel, EmailStr, field_validator
from typing import Optional
from datetime import datetime

class UserCreate(BaseModel):
    """注册请求"""
    username: str
    password: str
    email: Optional[str] = None
    
    @field_validator('username')
    def username_valid(cls, v):
        if len(v) < 3 or len(v) > 20:
            raise ValueError('用户名长度必须在3~20字符之间')
        if not v.isalnum():
            raise ValueError('用户名只能包含字母和数字')
        return v
    
    @field_validator('password')
    def password_valid(cls, v):
        if len(v) < 6:
            raise ValueError('密码至少6位')
        return v

class UserUpdate(BaseModel):
    """更新用户信息"""
    email: Optional[str] = None
    role: Optional[str] = None
    is_active: Optional[int] = None

class UserResponse(BaseModel):
    """用户响应（不包含密码）"""
    id: int
    username: str
    email: Optional[str]
    role: str
    is_active: int
    created_at: datetime
    
    class Config:
        from_attributes = True  # 允许从 SQLAlchemy 对象转换
```

---

## 用户 CRUD 接口完整实现

```python
# app/api/user.py（补充 CRUD 部分）
from fastapi import APIRouter, Depends, HTTPException, Query
from sqlalchemy.orm import Session
from app.core.database import get_db
from app.models.models import User
from app.schemas.user import UserCreate, UserUpdate, UserResponse
from app.services.auth import hash_password, require_admin, get_current_user
from typing import List

router = APIRouter()

@router.get("/", response_model=dict)
def list_users(
    page: int = Query(1, ge=1),
    page_size: int = Query(10, ge=1, le=100),
    keyword: str = Query(None),
    db: Session = Depends(get_db),
    _: User = Depends(require_admin)  # 仅管理员可访问
):
    """获取用户列表（分页 + 搜索）"""
    query = db.query(User)
    if keyword:
        query = query.filter(User.username.like(f"%{keyword}%"))
    
    total = query.count()
    users = query.offset((page - 1) * page_size).limit(page_size).all()
    
    return {
        "total": total,
        "items": [UserResponse.model_validate(u) for u in users],
        "page": page,
        "page_size": page_size
    }

@router.get("/{user_id}", response_model=UserResponse)
def get_user(
    user_id: int,
    db: Session = Depends(get_db),
    _: User = Depends(require_admin)
):
    """获取指定用户"""
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="用户不存在")
    return user

@router.put("/{user_id}", response_model=UserResponse)
def update_user(
    user_id: int,
    req: UserUpdate,
    db: Session = Depends(get_db),
    _: User = Depends(require_admin)
):
    """更新用户信息"""
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="用户不存在")
    
    update_data = req.model_dump(exclude_unset=True)
    for key, value in update_data.items():
        setattr(user, key, value)
    
    db.commit()
    db.refresh(user)
    return user

@router.delete("/{user_id}")
def delete_user(
    user_id: int,
    db: Session = Depends(get_db),
    current_user: User = Depends(require_admin)
):
    """禁用用户（软删除）"""
    if user_id == current_user.id:
        raise HTTPException(status_code=400, detail="不能禁用自己")
    
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="用户不存在")
    
    user.is_active = 0  # 软删除：标记为禁用
    db.commit()
    return {"message": f"用户 {user.username} 已禁用"}
```

---

## 为什么用软删除？

直接 `DELETE FROM users WHERE id=1` 有几个问题：
1. **数据丢失**：删除后无法追溯历史问答记录的归属
2. **外键约束**：qa_records 有 user_id 外键，直接删除会失败
3. **误操作**：真实业务里，删除往往是「无法挽回的」，而禁用是可逆的

软删除通过 `is_active=0` 标记用户为禁用状态，不删除数据，随时可以恢复。

---

## 本节小结

- Pydantic Schema 定义了接口的输入/输出格式，并提供数据验证
- `response_model` 确保返回数据不会泄露敏感字段（如密码）
- `model_dump(exclude_unset=True)` 只更新客户端传来的字段
- 软删除比物理删除更安全，是生产环境的最佳实践

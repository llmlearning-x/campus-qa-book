# 2.5 数据库设计

## 好的数据库设计，是系统稳定的基础

数据库设计做得好，后期修改成本很低；做得差，每次加新功能都要打补丁，越改越乱。

我们的系统数据库设计遵循以下原则：
1. **第三范式（3NF）**：消除冗余，每个字段只依赖主键
2. **软删除**：不真正删除记录，而是标记 `is_deleted=1`，保留数据可追溯性
3. **时间戳字段**：所有表都有 `created_at` 和 `updated_at`
4. **合理索引**：在查询频繁的字段上建索引，避免全表扫描

---

## ER 图

```
users (用户表)
  id, username, email, password, role, is_active
  |
  | 1 : N
  |
documents (文档表)
  id, filename, file_path, file_size, file_type,
  status, chunk_count, uploaded_by → users.id
  |
  | 1 : N
  |
document_chunks (文档切片表)
  id, doc_id → documents.id, chunk_index, content, vector_id

qa_records (问答记录表)
  id, user_id → users.id, session_id,
  question, answer, source_chunks
```

---

## 完整数据字典

### users — 用户表

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | BIGINT | PK, AUTO_INCREMENT | 用户ID |
| username | VARCHAR(50) | NOT NULL, UNIQUE | 用户名 |
| email | VARCHAR(100) | UNIQUE | 邮箱 |
| password | VARCHAR(255) | NOT NULL | bcrypt 哈希后的密码 |
| role | ENUM | DEFAULT 'user' | 角色：admin/user |
| is_active | TINYINT(1) | DEFAULT 1 | 1=启用，0=禁用 |
| created_at | DATETIME | DEFAULT NOW | 创建时间 |
| updated_at | DATETIME | ON UPDATE NOW | 更新时间 |

### documents — 文档表

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | BIGINT | PK | 文档ID |
| filename | VARCHAR(255) | NOT NULL | 原始文件名 |
| file_path | VARCHAR(500) | NOT NULL | 服务器存储路径 |
| file_size | BIGINT | | 文件大小（字节） |
| file_type | VARCHAR(50) | | pdf/txt/docx |
| status | ENUM | DEFAULT 'pending' | pending/processing/done/failed |
| chunk_count | INT | DEFAULT 0 | 切分后片段数 |
| uploaded_by | BIGINT | FK→users.id | 上传者ID |

### document_chunks — 文档切片表

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | BIGINT | PK | 切片ID |
| doc_id | BIGINT | FK→documents.id | 所属文档 |
| chunk_index | INT | NOT NULL | 切片序号（0开始） |
| content | TEXT | NOT NULL | 切片文本 |
| vector_id | BIGINT | | 在FAISS中的索引位置 |

### qa_records — 问答记录表

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | BIGINT | PK | 记录ID |
| user_id | BIGINT | FK→users.id | 提问用户（可空） |
| session_id | VARCHAR(100) | | 会话ID |
| question | TEXT | NOT NULL | 用户问题 |
| answer | TEXT | | AI回答 |
| source_chunks | JSON | | 来源切片ID列表 |
| created_at | DATETIME | DEFAULT NOW | 提问时间 |

---

## SQLAlchemy 模型定义

将以上表结构映射为 Python 对象 `app/models/models.py`：

```python
from datetime import datetime
from sqlalchemy import Column, BigInteger, String, Text, Integer, \
    DateTime, Enum, ForeignKey, JSON, TINYINT
from sqlalchemy.orm import relationship
from app.core.database import Base

class User(Base):
    __tablename__ = "users"
    
    id         = Column(BigInteger, primary_key=True, autoincrement=True)
    username   = Column(String(50), unique=True, nullable=False)
    email      = Column(String(100), unique=True)
    password   = Column(String(255), nullable=False)
    role       = Column(Enum("admin", "user"), default="user")
    is_active  = Column(TINYINT(1), default=1)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    documents  = relationship("Document", back_populates="uploader")
    qa_records = relationship("QARecord", back_populates="user")


class Document(Base):
    __tablename__ = "documents"
    
    id          = Column(BigInteger, primary_key=True, autoincrement=True)
    filename    = Column(String(255), nullable=False)
    file_path   = Column(String(500), nullable=False)
    file_size   = Column(BigInteger)
    file_type   = Column(String(50))
    status      = Column(
        Enum("pending", "processing", "done", "failed"), default="pending"
    )
    chunk_count = Column(Integer, default=0)
    uploaded_by = Column(BigInteger, ForeignKey("users.id"))
    created_at  = Column(DateTime, default=datetime.utcnow)
    updated_at  = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    uploader = relationship("User", back_populates="documents")
    chunks   = relationship("DocumentChunk", back_populates="document",
                            cascade="all, delete-orphan")


class DocumentChunk(Base):
    __tablename__ = "document_chunks"
    
    id          = Column(BigInteger, primary_key=True, autoincrement=True)
    doc_id      = Column(BigInteger, ForeignKey("documents.id"), nullable=False)
    chunk_index = Column(Integer, nullable=False)
    content     = Column(Text, nullable=False)
    vector_id   = Column(BigInteger)
    created_at  = Column(DateTime, default=datetime.utcnow)
    
    document = relationship("Document", back_populates="chunks")


class QARecord(Base):
    __tablename__ = "qa_records"
    
    id            = Column(BigInteger, primary_key=True, autoincrement=True)
    user_id       = Column(BigInteger, ForeignKey("users.id"))
    session_id    = Column(String(100))
    question      = Column(Text, nullable=False)
    answer        = Column(Text)
    source_chunks = Column(JSON)
    created_at    = Column(DateTime, default=datetime.utcnow)
    
    user = relationship("User", back_populates="qa_records")
```

---

## 本节小结

- 4 张核心表构成系统的数据基础
- 遵循 3NF、软删除、时间戳等设计原则
- SQLAlchemy ORM 将表映射为 Python 类，类型安全
- `cascade="all, delete-orphan"` 确保文档删除时切片数据自动清理

# 2.1 数据库环境搭建

## 连接 MySQL

如果你已经按照第1章完成了 MySQL 安装，我们现在来连接并验证。

推荐使用 **DBeaver**（免费、跨平台的数据库管理工具）进行可视化操作：

1. 下载 [DBeaver Community](https://dbeaver.io/download/)
2. 新建连接 → 选择 MySQL
3. 填写：Host `localhost`，Port `3306`，User `campus`，Password `campus123456`
4. 测试连接，看到 "Connected" 即成功

---

## 项目数据库初始化

运行以下 SQL 脚本初始化数据库（保存为 `init.sql`）：

```sql
-- 使用项目数据库
USE campus_qa;

-- 用户表
CREATE TABLE IF NOT EXISTS users (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    username    VARCHAR(50) NOT NULL UNIQUE COMMENT '用户名',
    email       VARCHAR(100) UNIQUE COMMENT '邮箱',
    password    VARCHAR(255) NOT NULL COMMENT '密码哈希',
    role        ENUM('admin', 'user') DEFAULT 'user' COMMENT '角色',
    is_active   TINYINT(1) DEFAULT 1 COMMENT '是否启用',
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at  DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_username (username),
    INDEX idx_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表';

-- 文档表
CREATE TABLE IF NOT EXISTS documents (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    filename        VARCHAR(255) NOT NULL COMMENT '原始文件名',
    file_path       VARCHAR(500) NOT NULL COMMENT '服务器存储路径',
    file_size       BIGINT COMMENT '文件大小（字节）',
    file_type       VARCHAR(50) COMMENT '文件类型（pdf/txt/docx）',
    status          ENUM('pending', 'processing', 'done', 'failed')
                    DEFAULT 'pending' COMMENT '处理状态',
    chunk_count     INT DEFAULT 0 COMMENT '切分后的片段数量',
    uploaded_by     BIGINT COMMENT '上传者ID',
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (uploaded_by) REFERENCES users(id),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='文档表';

-- 文档切片表（存储切分后的文本片段）
CREATE TABLE IF NOT EXISTS document_chunks (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    doc_id      BIGINT NOT NULL COMMENT '所属文档ID',
    chunk_index INT NOT NULL COMMENT '切片序号',
    content     TEXT NOT NULL COMMENT '切片文本内容',
    vector_id   BIGINT COMMENT 'FAISS中的向量索引ID',
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (doc_id) REFERENCES documents(id) ON DELETE CASCADE,
    INDEX idx_doc_id (doc_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='文档切片表';

-- 问答记录表
CREATE TABLE IF NOT EXISTS qa_records (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id         BIGINT COMMENT '提问用户ID（可为空，支持匿名）',
    session_id      VARCHAR(100) COMMENT '会话ID',
    question        TEXT NOT NULL COMMENT '用户问题',
    answer          TEXT COMMENT 'AI回答',
    source_chunks   JSON COMMENT '回答所依据的文档片段ID列表',
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id),
    INDEX idx_user_id (user_id),
    INDEX idx_session (session_id),
    INDEX idx_created (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='问答记录表';

-- 插入默认管理员账号（密码: admin123，实际使用时应修改）
INSERT IGNORE INTO users (username, email, password, role)
VALUES ('admin', 'admin@campus.edu', 
        '$2b$12$placeholder_hash_change_this', 'admin');
```

---

## 在 Python 中操作 MySQL

我们使用 **SQLAlchemy** ORM 来操作数据库，避免直接拼接 SQL 字符串（防注入、更安全）。

**安装依赖**：
```bash
pip install sqlalchemy pymysql
```

**数据库连接配置** `app/core/database.py`：
```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from app.core.config import settings

# 创建数据库引擎
engine = create_engine(
    settings.DATABASE_URL,
    pool_pre_ping=True,      # 自动检测连接是否有效
    pool_size=10,            # 连接池大小
    max_overflow=20,         # 超出 pool_size 后最多额外创建多少连接
    echo=False               # True 时打印所有 SQL（调试用）
)

# Session 工厂
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# 所有模型的基类
Base = declarative_base()

def get_db():
    """FastAPI 依赖注入：获取数据库会话"""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

---

## 本节小结

- DBeaver 是方便的数据库管理工具，建议安装
- 4 张核心表：users / documents / document_chunks / qa_records
- SQLAlchemy 提供 ORM 能力，避免直接拼接 SQL
- `get_db()` 是 FastAPI 依赖注入的标准用法

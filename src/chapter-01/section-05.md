# 1.5 开发环境搭建

## 为什么环境搭建很重要？

"在我电脑上能跑"是程序员界最臭名昭著的一句话。

环境不一致会导致代码在某台电脑上能运行，在另一台上报各种奇怪的错误。在团队开发中，统一环境是协作的前提。

本节的目标是让所有团队成员的开发环境完全一致。请**逐步执行**，遇到错误不要跳过。

---

## 必装软件清单

| 软件 | 用途 | 版本要求 |
|------|------|---------|
| Python | 后端开发语言 | 3.10+ |
| Node.js | 前端开发运行时 | 18+ |
| MySQL | 关系型数据库 | 8.0 |
| Git | 版本控制 | 最新 |
| VSCode | 代码编辑器 | 最新 |

---

## Step 1：安装 Python

### macOS
```bash
# 推荐使用 pyenv 管理 Python 版本
brew install pyenv
pyenv install 3.11.8
pyenv global 3.11.8

# 验证
python --version    # Python 3.11.8
```

### Windows
1. 访问 [python.org/downloads](https://www.python.org/downloads/)
2. 下载 Python 3.11.x（注意勾选 "Add Python to PATH"）
3. 验证：
```cmd
python --version    # Python 3.11.x
pip --version
```

### Linux (Ubuntu/Debian)
```bash
sudo apt update
sudo apt install python3.11 python3.11-venv python3-pip
```

---

## Step 2：安装 Node.js

推荐使用 nvm（Node Version Manager）管理 Node 版本：

### macOS / Linux
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc    # 或 source ~/.zshrc

nvm install 20
nvm use 20

# 验证
node --version    # v20.x.x
npm --version     # 10.x.x
```

### Windows
下载 [Node.js LTS 安装包](https://nodejs.org/zh-cn)，选择 20.x LTS 版本。

---

## Step 3：安装 MySQL

### macOS
```bash
brew install mysql@8.0
brew services start mysql
mysql_secure_installation    # 设置 root 密码
```

### Windows
1. 下载 [MySQL Installer](https://dev.mysql.com/downloads/installer/)
2. 选择 "Developer Default" 安装
3. 设置 root 密码，记住它！

### Linux
```bash
sudo apt install mysql-server
sudo systemctl start mysql
sudo mysql_secure_installation
```

**验证安装**：
```bash
mysql -u root -p
# 输入密码后进入 MySQL Shell
mysql> SELECT VERSION();    # 8.0.x
mysql> exit
```

---

## Step 4：创建项目数据库

```sql
-- 登录 MySQL 后执行
CREATE DATABASE campus_qa CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'campus'@'localhost' IDENTIFIED BY 'campus123456';
GRANT ALL PRIVILEGES ON campus_qa.* TO 'campus'@'localhost';
FLUSH PRIVILEGES;

-- 验证
SHOW DATABASES;    -- 应该能看到 campus_qa
```

---

## Step 5：后端项目初始化

```bash
# 创建项目目录
mkdir campus-qa-backend && cd campus-qa-backend

# 创建虚拟环境（重要！隔离项目依赖）
python -m venv venv

# 激活虚拟环境
source venv/bin/activate          # macOS/Linux
# 或
.\venv\Scripts\activate           # Windows

# 安装核心依赖
pip install fastapi uvicorn sqlalchemy pymysql \
    python-jose passlib bcrypt python-multipart \
    openai faiss-cpu langchain pydantic-settings

# 冻结依赖版本（方便队友复现）
pip freeze > requirements.txt
```

**项目目录结构**：
```
campus-qa-backend/
├── venv/              # 虚拟环境（不提交到 Git）
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── api/
│   ├── models/
│   ├── schemas/
│   ├── services/
│   └── core/
├── requirements.txt
└── .env               # 环境变量（不提交到 Git）
```

**创建 `.env` 文件**（存放敏感配置）：
```ini
# .env
DATABASE_URL=mysql+pymysql://campus:campus123456@localhost/campus_qa
SECRET_KEY=your-secret-key-change-this-in-production
OPENAI_API_KEY=sk-xxxxxxxxxxxx
```

> ⚠️ **安全警告**：`.env` 文件绝对不能提交到 Git！记得在 `.gitignore` 里加上它。

---

## Step 6：前端项目初始化

```bash
# 使用 Vite 创建 Vue3 项目
npm create vite@latest campus-qa-frontend -- --template vue
cd campus-qa-frontend

# 安装依赖
npm install

# 额外安装常用库
npm install element-plus @element-plus/icons-vue
npm install axios pinia vue-router
npm install -D @types/node

# 启动开发服务器
npm run dev
# 访问 http://localhost:5173 验证
```

---

## Step 7：环境验证脚本

保存以下脚本为 `check_env.py`，运行后应全部显示 ✅：

```python
#!/usr/bin/env python3
"""开发环境验证脚本"""

import sys
import subprocess

def check(name, cmd, expected_prefix):
    try:
        result = subprocess.run(
            cmd, capture_output=True, text=True, shell=True
        )
        version = result.stdout.strip()
        if version.startswith(expected_prefix):
            print(f"✅ {name}: {version}")
            return True
        else:
            print(f"❌ {name}: 版本不符合要求，当前 {version}")
            return False
    except Exception as e:
        print(f"❌ {name}: 未安装或无法检测 ({e})")
        return False

print("=== 开发环境检查 ===\n")

checks = [
    check("Python", "python --version", "Python 3.1"),
    check("pip",    "pip --version",    "pip"),
    check("Node",   "node --version",   "v"),
    check("npm",    "npm --version",    ""),
    check("Git",    "git --version",    "git"),
    check("MySQL",  "mysql --version",  "mysql"),
]

print("\n" + "=" * 20)
if all(checks):
    print("🎉 环境检查全部通过！可以开始开发了。")
else:
    failed = sum(1 for c in checks if not c)
    print(f"⚠️  有 {failed} 项检查未通过，请参照文档修复后再继续。")
```

运行：
```bash
python check_env.py
```

---

## VSCode 推荐插件

| 插件名 | 用途 |
|--------|------|
| Python | Python 语言支持 |
| Pylance | Python 智能补全 |
| Volar | Vue3 语言支持 |
| ESLint | JavaScript 代码规范 |
| Prettier | 代码格式化 |
| GitLens | Git 历史可视化 |
| Thunder Client | 接口测试（类似 Postman） |
| MySQL（by cweijan） | MySQL 可视化管理 |

---

## 常见问题

**Q: `pip install` 很慢怎么办？**
```bash
# 切换为阿里云镜像
pip install -i https://mirrors.aliyun.com/pypi/simple/ <包名>
# 或全局设置
pip config set global.index-url https://mirrors.aliyun.com/pypi/simple/
```

**Q: `npm install` 很慢怎么办？**
```bash
npm config set registry https://registry.npmmirror.com
```

**Q: MySQL 连接拒绝怎么办？**
```bash
# 检查 MySQL 服务是否启动
sudo systemctl status mysql        # Linux
brew services list | grep mysql   # macOS
```

---

## 本节小结

- 统一的开发环境是团队协作的前提
- Python 虚拟环境（venv）隔离项目依赖，必须使用
- `.env` 文件存放敏感配置，绝对不能提交到 Git
- 运行验证脚本确保所有工具正确安装

---

## 第1章小结

🎉 恭喜完成第一天的学习！

**你已经掌握了**：
- 软件开发的基本流程和敏捷开发理念
- 团队分工协作的角色与职责
- LLM、Embedding、RAG 的核心原理
- 系统整体架构的全局认知
- 完整的开发环境搭建

**第一天的产出物**：
- [ ] 人员分组表（含角色分配）
- [ ] 系统架构草图
- [ ] 开发环境验证脚本通过

**明天（Day 2）** 我们将真正开始写代码：搭建前后端脚手架、配置数据库、创建第一个 API 接口、跑通前后端联调。

---

_第1章完_

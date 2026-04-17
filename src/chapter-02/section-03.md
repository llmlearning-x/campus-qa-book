# 2.3 代码管理：GitLab 实战

## 为什么选 GitLab？

GitHub 是全球最大的代码托管平台，但很多企业出于数据安全考虑，会在内部自部署 GitLab。学习 GitLab 让你快速适应企业开发环境。

GitLab CE（社区版）完全免费，功能与企业版基本一致。

---

## GitLab 账号与项目创建

### 注册账号
1. 访问你们的 GitLab 实例（老师提供地址，如 `http://gitlab.campus.edu`）
2. 注册账号，用真实姓名或学号命名（便于老师识别）
3. 上传头像，设置 SSH Key（推荐）

### 设置 SSH Key（免密码 Push）

```bash
# 生成 SSH Key
ssh-keygen -t ed25519 -C "你的邮箱@example.com"
# 一路回车（使用默认路径和空密码）

# 查看公钥
cat ~/.ssh/id_ed25519.pub
```

将公钥内容复制到 GitLab → 头像 → Settings → SSH Keys。

### 创建项目仓库

1. 点击 "New Project" → "Create blank project"
2. 项目名：`campus-qa-backend` 和 `campus-qa-frontend`
3. 可见性选 `Internal`（组内成员可见）
4. 勾选 "Initialize repository with a README"

---

## 初始化本地项目并推送

### 后端项目

```bash
cd campus-qa-backend

# 初始化 .gitignore
cat > .gitignore << 'EOF'
# Python
__pycache__/
*.py[cod]
venv/
.env
*.egg-info/

# 上传文件
uploads/

# FAISS 向量数据
*.faiss
*.index

# IDE
.vscode/
.idea/
EOF

# 初始化 Git
git init
git add .
git commit -m "chore: 初始化后端项目结构"

# 关联远程仓库（替换为你的实际地址）
git remote add origin git@gitlab.campus.edu:yourname/campus-qa-backend.git
git branch -M main
git push -u origin main
```

### 创建开发分支

```bash
# 创建 dev 分支（日常开发在 dev 上进行）
git checkout -b dev
git push -u origin dev
```

---

## 分支协作工作流

每个新功能，从 `dev` 创建功能分支：

```bash
# 赵六：负责用户模块
git checkout dev
git pull origin dev          # 先拉取最新代码
git checkout -b feature/user-auth

# 开发完成后
git add .
git commit -m "feat: 完成用户注册登录接口"
git push origin feature/user-auth

# 在 GitLab 上发起 Merge Request (MR)
# 指定队友 Review，通过后合并到 dev
```

---

## 每日工作流程

```bash
# 早上开始工作前：拉取最新代码
git pull origin dev

# 开发过程中：频繁小 commit
git add .
git commit -m "feat: 添加用户查询分页接口"

# 遇到冲突时
git status          # 查看冲突文件
# 手动解决冲突后
git add .
git commit -m "fix: 解决用户模块合并冲突"

# 下班前：推送代码
git push origin feature/user-auth
```

---

## 本节小结

- SSH Key 设置后可以免密码推送代码
- 分支策略：main（稳定）→ dev（开发）→ feature/xxx（功能）
- Commit 信息规范化便于追溯历史
- 每天开始工作先 pull，结束工作后 push

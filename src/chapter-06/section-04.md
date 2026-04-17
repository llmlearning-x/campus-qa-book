# 6.4 项目部署发布

## 后端部署

### 方式一：直接运行（适合演示环境）

```bash
# 在服务器上
cd campus-qa-backend
source venv/bin/activate
pip install -r requirements.txt

# 启动（后台运行）
nohup uvicorn app.main:app --host 0.0.0.0 --port 8000 > app.log 2>&1 &
echo $! > app.pid  # 保存进程ID

# 停止
kill $(cat app.pid)
```

### 方式二：supervisor 进程守护（推荐生产环境）

```bash
# 安装 supervisor
pip install supervisor

# 配置文件 /etc/supervisor/conf.d/campus-qa.conf
```

```ini
[program:campus-qa]
command=/path/to/venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8000
directory=/path/to/campus-qa-backend
user=ubuntu
autostart=true
autorestart=true
stdout_logfile=/var/log/campus-qa.log
stderr_logfile=/var/log/campus-qa-error.log
```

```bash
supervisorctl update
supervisorctl start campus-qa
supervisorctl status
```

---

## 前端部署

```bash
# 构建生产版本
cd campus-qa-frontend
npm run build
# 生成 dist/ 目录（包含 index.html 和静态资源）
```

### 用 Nginx 托管前端静态文件

```nginx
# /etc/nginx/sites-available/campus-qa
server {
    listen 80;
    server_name your-server-ip;

    # 前端静态文件
    location / {
        root /path/to/campus-qa-frontend/dist;
        try_files $uri $uri/ /index.html;  # SPA 必须加这行
        index index.html;
    }

    # 后端 API 反向代理
    location /api {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        # SSE 必须关闭缓冲
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 300s;
    }
}
```

```bash
nginx -t         # 检查配置语法
nginx -s reload  # 重新加载配置
```

---

## 环境变量安全管理

**绝对不要做的事**：
- ❌ 把 `.env` 提交到 Git
- ❌ 在代码里硬编码 API Key
- ❌ 在日志里打印密钥

**应该做的事**：
- ✅ 把 `.env` 加入 `.gitignore`
- ✅ 生产环境的密钥由运维通过安全渠道传递
- ✅ 定期轮换密钥

---

## 本节小结

- `uvicorn` + supervisor 是 Python 后端部署的常见组合
- Nginx 负责静态文件托管 + API 反向代理
- SSE 接口的 Nginx 必须关闭 `proxy_buffering`，否则流式输出会被缓冲
- 生产环境的 API Key 安全管理是基本职业素养

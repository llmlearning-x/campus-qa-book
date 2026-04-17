# 附录A：开发环境配置速查表

## 软件版本

| 软件 | 推荐版本 | 安装命令（macOS） |
|------|---------|----------------|
| Python | 3.11+ | `pyenv install 3.11.8` |
| Node.js | 20 LTS | `nvm install 20` |
| MySQL | 8.0 | `brew install mysql@8.0` |
| Git | 最新 | `brew install git` |

## pip 镜像（国内加速）
```bash
pip config set global.index-url https://mirrors.aliyun.com/pypi/simple/
```

## npm 镜像（国内加速）
```bash
npm config set registry https://registry.npmmirror.com
```

## 关键命令速查

```bash
# 后端启动
cd campus-qa-backend && source venv/bin/activate
uvicorn app.main:app --reload --port 8000

# 前端启动
cd campus-qa-frontend && npm run dev

# MySQL 连接
mysql -u campus -p campus123456 campus_qa

# 查看后端日志
tail -f app.log

# 重启后端服务
kill $(cat app.pid) && nohup uvicorn app.main:app ... > app.log &
```

# 5.1 本地开发环境搭建

## 1. 系统要求

| 组件 | 版本要求 | 说明 |
| :--- | :--- | :--- |
| **操作系统** | macOS / Linux / Windows (WSL2) | 推荐 macOS 或 Linux |
| **Python** | 3.11.x | 必须精确匹配 3.11 |
| **Node.js** | 18+ | 前端开发 |
| **pnpm** | 8+ | 前端包管理 |
| **PostgreSQL** | 16 + pgvector | 数据库 |
| **Docker** | 20+ | 可选，用于运行 PostgreSQL |

---

## 2. 快速开始

### 2.1 克隆代码

```bash
git clone https://github.com/dataease/SQLBot.git
cd SQLBot
```

### 2.2 启动 PostgreSQL

推荐使用 Docker 运行 PostgreSQL（含 pgvector）：

```bash
docker run -d \
  --name pg-docker \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -e POSTGRES_DB=sqlbot \
  -p 5432:5432 \
  pgvector/pgvector:pg16
```

### 2.3 配置环境变量

创建 `.env` 文件（项目根目录）：

```bash
# 数据库配置
POSTGRES_SERVER=127.0.0.1
POSTGRES_PORT=5432
POSTGRES_DB=sqlbot
POSTGRES_USER=postgres
POSTGRES_PASSWORD=mysecretpassword

# 或使用完整连接串
# SQLBOT_DB_URL=postgresql+psycopg://postgres:mysecretpassword@127.0.0.1:5432/sqlbot

# 可选配置
LOG_LEVEL=INFO
EMBEDDING_ENABLED=true
```

### 2.4 启动后端

```bash
cd backend

# 安装 uv (如未安装)
curl -LsSf https://astral.sh/uv/install.sh | sh

# 同步依赖
uv sync

# 运行数据库迁移
uv run alembic upgrade head

# 启动开发服务器
uv run uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

### 2.5 启动前端

```bash
cd frontend

# 安装依赖
pnpm install

# 启动开发服务器
pnpm dev
```

### 2.6 访问应用

- 前端地址：http://localhost:5173
- 后端 API：http://localhost:8000
- API 文档：http://localhost:8000/docs
- 默认账号：`admin` / `SQLBot@123456`

---

## 3. 详细配置

### 3.1 后端环境变量

| 变量名 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `POSTGRES_SERVER` | localhost | 数据库地址 |
| `POSTGRES_PORT` | 5432 | 数据库端口 |
| `POSTGRES_DB` | sqlbot | 数据库名 |
| `POSTGRES_USER` | root | 数据库用户 |
| `POSTGRES_PASSWORD` | - | 数据库密码 |
| `SQLBOT_DB_URL` | - | 完整数据库连接串（优先级高） |
| `LOG_LEVEL` | INFO | 日志级别 |
| `EMBEDDING_ENABLED` | true | 是否启用向量检索 |
| `CACHE_TYPE` | memory | 缓存类型 (memory/redis) |
| `CACHE_REDIS_URL` | - | Redis 连接串 |

### 3.2 前端环境变量

| 变量名 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `VITE_API_BASE_URL` | http://localhost:8000/api/v1 | 后端 API 地址 |
| `VITE_APP_TITLE` | SQLBot | 应用标题 |

---

## 4. IDE 配置

### 4.1 VS Code 推荐扩展

```json
// .vscode/extensions.json
{
  "recommendations": [
    "ms-python.python",
    "ms-python.vscode-pylance",
    "charliermarsh.ruff",
    "Vue.volar",
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode"
  ]
}
```

### 4.2 Python 开发配置

```json
// .vscode/settings.json
{
  "python.defaultInterpreterPath": "./backend/.venv/bin/python",
  "python.analysis.typeCheckingMode": "basic",
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true
  }
}
```

---

## 5. 常用开发命令

### 5.1 后端命令

```bash
# 进入后端目录
cd backend

# 运行测试
uv run pytest

# 代码格式化
uv run ruff format .

# 代码检查
uv run ruff check .

# 类型检查
uv run mypy .

# 创建数据库迁移
uv run alembic revision --autogenerate -m "描述"

# 执行迁移
uv run alembic upgrade head
```

### 5.2 前端命令

```bash
# 进入前端目录
cd frontend

# 开发模式
pnpm dev

# 生产构建
pnpm build

# 代码检查
pnpm lint

# 预览构建结果
pnpm preview
```

---

## 6. 调试技巧

### 6.1 后端调试

1. **VS Code 调试配置**：

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "FastAPI",
      "type": "debugpy",
      "request": "launch",
      "module": "uvicorn",
      "args": ["main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"],
      "cwd": "${workspaceFolder}/backend",
      "envFile": "${workspaceFolder}/.env"
    }
  ]
}
```

2. **SQL 调试**：设置 `SQL_DEBUG=true` 查看 SQLAlchemy 执行的 SQL。

### 6.2 前端调试

1. 使用 Vue DevTools 浏览器扩展
2. 在 Chrome DevTools 的 Sources 面板设置断点

---

## 7. 常见问题

### Q1: 数据库连接失败

**症状**：`connection refused` 或认证失败

**解决方案**：
1. 确认 PostgreSQL 正在运行：`docker ps`
2. 检查 `.env` 中的用户名密码是否正确
3. 确认数据库 `sqlbot` 已创建

### Q2: Embedding 模型下载失败

**症状**：`FileNotFoundError: Path ... not found`

**解决方案**：
1. 确保网络可访问 HuggingFace
2. 或手动下载模型到 `backend/models/embedding/` 目录

### Q3: 前端 API 请求 404

**症状**：前端报 Network Error

**解决方案**：
1. 确认后端已启动
2. 检查 `frontend/.env.development` 中的 API 地址
3. 确认端口没有被占用

### Q4: 端口被占用

**解决方案**：
```bash
# 查看端口占用
lsof -i :8000

# 或使用其他端口启动
uv run uvicorn main:app --port 8010
```

---

## 8. 项目结构速查

```
SQLBot/
├── .env                    # 环境变量配置
├── docker-compose.yaml     # Docker 编排
├── backend/
│   ├── main.py             # 后端入口
│   ├── apps/               # 业务模块
│   ├── common/             # 公共模块
│   ├── alembic/            # 数据库迁移
│   └── pyproject.toml      # Python 依赖
├── frontend/
│   ├── src/                # 前端源码
│   ├── package.json        # 前端依赖
│   └── vite.config.ts      # Vite 配置
└── g2-ssr/                 # 图表渲染服务
```

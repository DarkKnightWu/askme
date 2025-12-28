# 5.2 Docker 部署指南

## 1. 快速部署

### 1.1 单容器部署（推荐入门）

```bash
docker run -d \
  --name sqlbot \
  --restart unless-stopped \
  -p 8000:8000 \
  -p 8001:8001 \
  -v ./data/sqlbot/excel:/opt/sqlbot/data/excel \
  -v ./data/sqlbot/file:/opt/sqlbot/data/file \
  -v ./data/sqlbot/images:/opt/sqlbot/images \
  -v ./data/sqlbot/logs:/opt/sqlbot/app/logs \
  -v ./data/postgresql:/var/lib/postgresql/data \
  --privileged=true \
  dataease/sqlbot
```

### 1.2 访问应用

- 地址：http://your-server-ip:8000
- 账号：`admin`
- 密码：`SQLBot@123456`

---

## 2. Docker Compose 部署

### 2.1 docker-compose.yaml

```yaml
version: '3.8'

services:
  sqlbot:
    image: dataease/sqlbot:latest
    container_name: sqlbot
    restart: unless-stopped
    networks:
      - sqlbot-network
    ports:
      - "8000:8000"   # Web UI & API
      - "8001:8001"   # MCP Server
    environment:
      # 数据库配置 (内置 PostgreSQL)
      POSTGRES_SERVER: localhost
      POSTGRES_PORT: 5432
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-SQLBot@pg123}
      POSTGRES_DB: sqlbot
      
      # 日志配置
      LOG_LEVEL: ${LOG_LEVEL:-INFO}
      
      # AI 配置
      EMBEDDING_ENABLED: ${EMBEDDING_ENABLED:-true}
      
      # 缓存配置
      CACHE_TYPE: ${CACHE_TYPE:-memory}
    volumes:
      # 数据持久化
      - ./data/postgresql:/var/lib/postgresql/data
      - ./data/excel:/opt/sqlbot/data/excel
      - ./data/file:/opt/sqlbot/data/file
      - ./data/images:/opt/sqlbot/images
      - ./data/logs:/opt/sqlbot/app/logs
      - ./data/models:/opt/sqlbot/models
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/api/v1/settings/basic"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  sqlbot-network:
    driver: bridge
```

### 2.2 启动服务

```bash
# 启动
docker compose up -d

# 查看日志
docker compose logs -f

# 停止
docker compose down

# 重启
docker compose restart
```

---

## 3. 外置数据库部署

### 3.1 使用外部 PostgreSQL

```yaml
# docker-compose.yaml
services:
  sqlbot:
    image: dataease/sqlbot:latest
    environment:
      # 使用外部数据库
      SQLBOT_DB_URL: postgresql+psycopg://user:password@db-host:5432/sqlbot
    # 不需要挂载 postgresql 数据卷
    volumes:
      - ./data/excel:/opt/sqlbot/data/excel
      - ./data/file:/opt/sqlbot/data/file
      - ./data/images:/opt/sqlbot/images
      - ./data/logs:/opt/sqlbot/app/logs

  # 可选：单独部署 PostgreSQL
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: sqlbot
      POSTGRES_USER: sqlbot
      POSTGRES_PASSWORD: your-secure-password
    volumes:
      - ./data/postgresql:/var/lib/postgresql/data
    ports:
      - "5432:5432"
```

> **注意**：外部 PostgreSQL 必须安装 pgvector 扩展。

---

## 4. 环境变量参考

| 变量 | 默认值 | 说明 |
| :--- | :--- | :--- |
| **数据库** |||
| `POSTGRES_SERVER` | localhost | 数据库地址 |
| `POSTGRES_PORT` | 5432 | 数据库端口 |
| `POSTGRES_DB` | sqlbot | 数据库名 |
| `POSTGRES_USER` | postgres | 用户名 |
| `POSTGRES_PASSWORD` | - | 密码 |
| `SQLBOT_DB_URL` | - | 完整连接串 (优先级高) |
| **应用** |||
| `LOG_LEVEL` | INFO | 日志级别 |
| `DEFAULT_PWD` | SQLBot@123456 | 默认密码 |
| **AI** |||
| `EMBEDDING_ENABLED` | true | 启用向量检索 |
| `LOCAL_MODEL_PATH` | /opt/sqlbot/models | 模型路径 |
| **缓存** |||
| `CACHE_TYPE` | memory | 缓存类型 |
| `CACHE_REDIS_URL` | - | Redis 地址 |

---

## 5. 数据卷说明

| 容器路径 | 说明 | 必须持久化 |
| :--- | :--- | :---: |
| `/var/lib/postgresql/data` | PostgreSQL 数据 | ✅ |
| `/opt/sqlbot/data/excel` | 上传的 Excel 文件 | ✅ |
| `/opt/sqlbot/data/file` | 上传的其他文件 | ✅ |
| `/opt/sqlbot/images` | 生成的图表图片 | ⚠️ |
| `/opt/sqlbot/app/logs` | 应用日志 | ⚠️ |
| `/opt/sqlbot/models` | AI 模型文件 | ⚠️ |

---

## 6. 网络配置

### 6.1 端口映射

| 端口 | 用途 |
| :---: | :--- |
| 8000 | Web UI 和 REST API |
| 8001 | MCP Server |

### 6.2 反向代理配置 (Nginx)

```nginx
server {
    listen 80;
    server_name sqlbot.example.com;
    
    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # SSE 支持
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 86400s;
    }
}
```

---

## 7. 健康检查

### 7.1 检查端点

```bash
# 健康检查
curl http://localhost:8000/api/v1/settings/basic

# 预期响应
{"code": 200, "data": {...}}
```

### 7.2 Docker 健康状态

```bash
docker inspect --format='{{.State.Health.Status}}' sqlbot
```

---

## 8. 日志管理

### 8.1 查看实时日志

```bash
# 容器日志
docker logs -f sqlbot

# 应用日志
docker exec sqlbot tail -f /opt/sqlbot/app/logs/sqlbot.log
```

### 8.2 日志级别调整

```bash
# 通过环境变量
docker run -e LOG_LEVEL=DEBUG ...

# 或在 docker-compose.yaml 中
environment:
  LOG_LEVEL: DEBUG
```

---

## 9. 升级指南

### 9.1 备份数据

```bash
# 备份 PostgreSQL
docker exec sqlbot pg_dump -U postgres sqlbot > backup.sql

# 备份数据卷
tar -czvf sqlbot-data.tar.gz ./data/
```

### 9.2 升级步骤

```bash
# 拉取新镜像
docker pull dataease/sqlbot:latest

# 停止旧容器
docker compose down

# 启动新容器 (数据卷保持不变)
docker compose up -d

# 验证
docker logs -f sqlbot
```

---

## 10. 常见问题

### Q1: 容器启动后无法访问

**检查项**：
1. 防火墙是否开放 8000 端口
2. 容器是否健康：`docker ps`
3. 查看日志：`docker logs sqlbot`

### Q2: 数据库初始化失败

**解决方案**：
1. 确保 PostgreSQL 数据卷为空（首次部署）
2. 或检查 `SQLBOT_DB_URL` 格式是否正确

### Q3: 内存不足

**推荐配置**：
- 最小：4GB RAM
- 推荐：8GB RAM（启用 Embedding）

```bash
# 限制容器内存
docker run --memory=4g ...
```

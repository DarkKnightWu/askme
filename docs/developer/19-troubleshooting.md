# 5.4 故障排查手册

## 1. 快速诊断清单

| 症状 | 可能原因 | 快速检查 |
| :--- | :--- | :--- |
| 无法访问页面 | 服务未启动/端口冲突 | `docker ps` 或 `lsof -i:8000` |
| 登录失败 | 密码错误/Token 过期 | 检查凭据/清除缓存 |
| SQL 生成失败 | AI 模型未配置/连接超时 | 检查 AI 模型配置 |
| 查询无结果 | 数据源连接失败 | 测试数据源连接 |
| Embedding 报错 | 模型文件缺失 | 检查模型目录 |

---

## 2. 后端问题排查

### 2.1 服务启动失败

#### 症状
```
uvicorn: command not found
```

#### 解决方案
```bash
# 确保在虚拟环境中
cd backend
source .venv/bin/activate  # 或使用 uv run

# 或直接使用 uv
uv run uvicorn main:app --port 8000
```

---

#### 症状
```
ModuleNotFoundError: No module named 'xxx'
```

#### 解决方案
```bash
cd backend
uv sync  # 重新同步依赖
```

---

#### 症状
```
sqlalchemy.exc.OperationalError: connection refused
```

#### 解决方案
1. 检查数据库是否运行：
```bash
docker ps | grep postgres
```

2. 检查连接配置：
```bash
cat ../.env | grep POSTGRES
```

3. 测试连接：
```bash
docker exec -it pg-docker psql -U postgres -d sqlbot -c "SELECT 1"
```

---

### 2.2 数据库迁移问题

#### 症状
```
alembic.util.exc.CommandError: Can't locate revision identified by 'xxx'
```

#### 解决方案
```bash
# 查看当前版本
uv run alembic current

# 查看历史
uv run alembic history

# 强制设置版本（谨慎使用）
uv run alembic stamp head
```

---

#### 症状
```
psycopg2.errors.UndefinedTable: relation "xxx" does not exist
```

#### 解决方案
```bash
# 执行迁移
uv run alembic upgrade head
```

---

### 2.3 AI 模型问题

#### 症状
```
FileNotFoundError: Path ./models/embedding/... not found
```

#### 原因
Embedding 模型未下载

#### 解决方案
1. 确保网络可访问 HuggingFace
2. 或手动下载模型到 `backend/models/embedding/` 目录
3. 重启服务，模型会自动下载

---

#### 症状
```
openai.AuthenticationError: Incorrect API key provided
```

#### 解决方案
1. 进入「系统管理 → AI 模型」
2. 检查 API 密钥是否正确
3. 确认账户余额充足

---

#### 症状
```
Connection timeout / Connection refused (LLM 调用)
```

#### 解决方案
1. 检查 API 地址是否正确
2. 检查网络连通性：
```bash
curl -I https://api.deepseek.com
```
3. 检查是否需要代理

---

### 2.4 数据源连接问题

#### 症状
```
Connect DB failed
```

#### 排查步骤
1. 检查数据源配置（IP、端口、用户名、密码）
2. 从 SQLBot 服务器测试连接：
```bash
# MySQL
mysql -h <host> -P <port> -u <user> -p

# PostgreSQL
psql -h <host> -p <port> -U <user> -d <database>
```
3. 检查防火墙规则

---

## 3. 前端问题排查

### 3.1 页面加载问题

#### 症状
白屏或页面无法加载

#### 解决方案
1. 检查浏览器控制台错误
2. 清除浏览器缓存
3. 检查 API 地址配置：
```bash
cat frontend/.env.development
```

---

#### 症状
```
Network Error / 连接被拒绝
```

#### 解决方案
1. 确认后端服务正在运行
2. 检查 CORS 配置
3. 检查端口是否匹配

---

### 3.2 登录问题

#### 症状
登录后立即跳回登录页

#### 解决方案
1. 清除浏览器本地存储
2. 检查 Token 是否正确存储
3. 查看后端日志中的认证错误

---

### 3.3 SSE 流式响应问题

#### 症状
问答时没有流式输出，直接超时

#### 解决方案
1. 检查 Nginx 配置是否支持 SSE：
```nginx
proxy_buffering off;
proxy_cache off;
proxy_read_timeout 86400s;
```

2. 检查浏览器是否阻止 EventSource

---

## 4. Docker 问题排查

### 4.1 容器启动失败

#### 查看日志
```bash
docker logs sqlbot
```

#### 常见错误

**端口被占用**
```
Bind for 0.0.0.0:8000 failed: port is already allocated
```

解决方案：
```bash
# 查看占用进程
lsof -i :8000

# 或使用其他端口
docker run -p 8010:8000 ...
```

---

**数据卷权限问题**
```
PermissionError: [Errno 13] Permission denied
```

解决方案：
```bash
# 修改目录权限
chmod -R 777 ./data/
```

---

### 4.2 容器资源不足

#### 症状
容器频繁重启或 OOM Killed

#### 解决方案
```bash
# 查看资源使用
docker stats sqlbot

# 增加内存限制
docker run --memory=8g ...
```

---

## 5. 日志位置速查

| 日志类型 | 容器路径 | 宿主机路径(推荐挂载) |
| :--- | :--- | :--- |
| 应用日志 | /opt/sqlbot/app/logs | ./data/logs |
| PostgreSQL 日志 | /var/lib/postgresql/data/log | ./data/postgresql/log |

---

## 6. 诊断命令汇总

```bash
# 服务状态
docker ps -a | grep sqlbot
docker inspect --format='{{.State.Health.Status}}' sqlbot

# 查看日志
docker logs -f --tail 100 sqlbot

# 进入容器
docker exec -it sqlbot bash

# 检查数据库
docker exec sqlbot psql -U postgres -d sqlbot -c "SELECT count(*) FROM sys_user"

# 检查网络
docker exec sqlbot curl -I http://localhost:8000/api/v1/settings/basic

# 查看资源使用
docker stats sqlbot
```

---

## 7. 常见错误代码

| HTTP 状态码 | 含义 | 可能原因 |
| :---: | :--- | :--- |
| 401 | 未认证 | Token 过期或无效 |
| 403 | 禁止访问 | 权限不足 |
| 404 | 未找到 | 资源不存在或 API 路径错误 |
| 500 | 服务器错误 | 后端异常，查看日志 |
| 502 | 网关错误 | 后端服务未启动 |
| 504 | 网关超时 | 请求处理超时 |

---

## 8. 获取帮助

### 8.1 提交 Issue

在 [GitHub Issues](https://github.com/dataease/SQLBot/issues) 提交问题时，请附上：

1. SQLBot 版本
2. 部署方式（Docker/源码）
3. 操作系统
4. 错误截图/日志
5. 复现步骤

### 8.2 社区支持

加入技术交流群获取实时帮助。

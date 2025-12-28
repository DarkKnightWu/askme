# 3.1 API 接口文档

## 1. 概述

SQLBot 后端提供 RESTful API，遵循以下规范：

| 规范项 | 说明 |
| :--- | :--- |
| **协议** | HTTP/HTTPS |
| **数据格式** | JSON |
| **认证方式** | JWT Token / API Key |
| **API 前缀** | `/api/v1` |
| **文档地址** | `/docs` (Swagger UI) |

---

## 2. 认证方式

### 2.1 JWT Token

```http
X-SQLBOT-TOKEN: <your_jwt_token>
```

### 2.2 API Key

```http
Authorization: Bearer <access_key>:<secret_key>
```

---

## 3. 响应格式

### 3.1 成功响应

```json
{
  "code": 200,
  "message": "success",
  "data": { ... }
}
```

### 3.2 错误响应

```json
{
  "code": 400,
  "message": "错误描述",
  "data": null
}
```

### 3.3 状态码说明

| 状态码 | 说明 |
| :---: | :--- |
| 200 | 成功 |
| 400 | 请求参数错误 |
| 401 | 未认证 |
| 403 | 权限不足 |
| 404 | 资源不存在 |
| 500 | 服务器错误 |

---

## 4. 核心 API

### 4.1 认证模块

#### 登录
```http
POST /api/v1/login
Content-Type: application/json

{
  "username": "admin",
  "password": "SQLBot@123456"
}
```

**响应**：
```json
{
  "code": 200,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIs...",
    "user": {
      "id": 1,
      "username": "admin",
      "role": 1
    }
  }
}
```

#### 获取当前用户
```http
GET /api/v1/user/me
X-SQLBOT-TOKEN: <token>
```

---

### 4.2 智能问答模块

#### 提问（流式）
```http
GET /api/v1/chat/question?question=上个月销售额是多少&ds_id=1&model_id=1
X-SQLBOT-TOKEN: <token>
Accept: text/event-stream
```

**SSE 响应事件**：

| 事件类型 | 说明 |
| :--- | :--- |
| `sql-result` | SQL 生成进度 |
| `sql` | 最终 SQL |
| `sql-data` | 查询结果数据 |
| `chart` | 图表配置 |
| `finish` | 完成 |
| `error` | 错误信息 |

#### 获取对话列表
```http
GET /api/v1/chat/list?page=1&size=20
X-SQLBOT-TOKEN: <token>
```

#### 获取对话详情
```http
GET /api/v1/chat/{chat_id}
X-SQLBOT-TOKEN: <token>
```

#### 删除对话
```http
DELETE /api/v1/chat/{chat_id}
X-SQLBOT-TOKEN: <token>
```

---

### 4.3 数据源模块

#### 获取数据源列表
```http
GET /api/v1/datasource/list
X-SQLBOT-TOKEN: <token>
```

**响应**：
```json
{
  "code": 200,
  "data": [
    {
      "id": 1,
      "name": "业务数据库",
      "type": "mysql",
      "status": 1
    }
  ]
}
```

#### 创建数据源
```http
POST /api/v1/datasource
X-SQLBOT-TOKEN: <token>
Content-Type: application/json

{
  "name": "业务数据库",
  "type": "mysql",
  "host": "db.example.com",
  "port": 3306,
  "database": "business",
  "username": "readonly",
  "password": "password123"
}
```

#### 测试连接
```http
POST /api/v1/datasource/test
X-SQLBOT-TOKEN: <token>
Content-Type: application/json

{
  "type": "mysql",
  "host": "db.example.com",
  "port": 3306,
  "database": "business",
  "username": "readonly",
  "password": "password123"
}
```

#### 同步表结构
```http
POST /api/v1/datasource/{ds_id}/sync
X-SQLBOT-TOKEN: <token>
```

#### 获取表列表
```http
GET /api/v1/datasource/{ds_id}/tables
X-SQLBOT-TOKEN: <token>
```

---

### 4.4 用户管理模块

#### 获取用户列表
```http
GET /api/v1/user/list?page=1&size=20
X-SQLBOT-TOKEN: <token>
```

#### 创建用户
```http
POST /api/v1/user
X-SQLBOT-TOKEN: <token>
Content-Type: application/json

{
  "username": "newuser",
  "email": "newuser@example.com",
  "password": "Password123!",
  "role": 2
}
```

#### 修改密码
```http
PUT /api/v1/user/password
X-SQLBOT-TOKEN: <token>
Content-Type: application/json

{
  "old_password": "oldpass",
  "new_password": "newpass"
}
```

---

### 4.5 AI 模型模块

#### 获取模型列表
```http
GET /api/v1/aimodel/list
X-SQLBOT-TOKEN: <token>
```

#### 创建模型
```http
POST /api/v1/aimodel
X-SQLBOT-TOKEN: <token>
Content-Type: application/json

{
  "name": "GPT-4",
  "supplier": 1,
  "base_model": "gpt-4",
  "api_domain": "https://api.openai.com/v1",
  "api_key": "sk-xxx",
  "protocol": 1,
  "default_model": true
}
```

---

### 4.6 术语库模块

#### 获取术语列表
```http
GET /api/v1/terminology/list?page=1&size=20
X-SQLBOT-TOKEN: <token>
```

#### 创建术语
```http
POST /api/v1/terminology
X-SQLBOT-TOKEN: <token>
Content-Type: application/json

{
  "word": "GMV",
  "description": "商品交易总额，计算公式为 SUM(order_amount)"
}
```

---

### 4.7 工作空间模块

#### 获取工作空间列表
```http
GET /api/v1/workspace/list
X-SQLBOT-TOKEN: <token>
```

#### 切换工作空间
```http
POST /api/v1/workspace/switch/{workspace_id}
X-SQLBOT-TOKEN: <token>
```

---

## 5. MCP 接口

MCP (Model Context Protocol) 接口用于与 AI 应用集成。

### 5.1 获取数据源列表
```http
GET /api/v1/mcp/datasource
```

### 5.2 获取模型列表
```http
GET /api/v1/mcp/model
```

### 5.3 提问
```http
POST /api/v1/mcp/question
Content-Type: application/json

{
  "question": "查询销售数据",
  "ds_id": 1,
  "model_id": 1
}
```

---

## 6. Swagger 文档

### 6.1 访问地址

```
http://localhost:8000/docs
```

### 6.2 多语言支持

```
http://localhost:8000/docs?lang=zh  # 中文
http://localhost:8000/docs?lang=en  # 英文
```

### 6.3 OpenAPI 规范

```
http://localhost:8000/openapi.json
```

---

## 7. API 限流

生产环境建议配置限流：

| 接口 | 限流策略 |
| :--- | :--- |
| `/api/v1/login` | 10 次/分钟/IP |
| `/api/v1/chat/question` | 60 次/分钟/用户 |
| 其他接口 | 100 次/分钟/用户 |

---

## 8. SDK 示例

### 8.1 Python

```python
import requests

class SQLBotClient:
    def __init__(self, base_url: str, token: str):
        self.base_url = base_url
        self.headers = {"X-SQLBOT-TOKEN": token}
    
    def ask(self, question: str, ds_id: int, model_id: int):
        url = f"{self.base_url}/api/v1/chat/question"
        params = {"question": question, "ds_id": ds_id, "model_id": model_id}
        
        with requests.get(url, params=params, headers=self.headers, stream=True) as r:
            for line in r.iter_lines():
                if line:
                    yield line.decode()

# 使用
client = SQLBotClient("http://localhost:8000", "your_token")
for event in client.ask("上个月销售额", ds_id=1, model_id=1):
    print(event)
```

### 8.2 JavaScript

```javascript
async function askQuestion(question, dsId, modelId) {
  const url = `/api/v1/chat/question?question=${encodeURIComponent(question)}&ds_id=${dsId}&model_id=${modelId}`;
  
  const eventSource = new EventSource(url);
  
  eventSource.onmessage = (event) => {
    const data = JSON.parse(event.data);
    console.log(data);
  };
  
  eventSource.onerror = () => {
    eventSource.close();
  };
}
```

# 3.2 核心模块详细设计

## 1. 概述

本文档详细描述 SQLBot 核心业务模块的设计，包括类图、时序图和关键实现。

---

## 2. Chat 模块（智能问答）

### 2.1 类结构

```mermaid
classDiagram
    class LLMService {
        +CoreDatasource ds
        +ChatQuestion chat_question
        +ChatRecord record
        +LLMConfig config
        +BaseChatModel llm
        +List~BaseMessage~ sql_message
        +List~BaseMessage~ chart_message
        +CurrentUser current_user
        +run_task()
        +generate_sql()
        +execute_sql()
        +generate_chart()
        +generate_analysis()
        +generate_predict()
    }
    
    class ChatQuestion {
        +str question
        +int ds_id
        +int model_id
        +int chart_type
        +sys_question() str
        +user_question() str
        +chart_sys_question() str
    }
    
    class ChatRecord {
        +int id
        +int chat_id
        +str question
        +str sql
        +str data
        +str chart
        +int status
    }
    
    LLMService --> ChatQuestion
    LLMService --> ChatRecord
    LLMService --> LLMConfig
```

### 2.2 问答流程时序图

```mermaid
sequenceDiagram
    participant User as 用户
    participant API as Chat API
    participant Service as LLMService
    participant LLM as LLM 模型
    participant DB as 业务数据库
    participant G2 as G2-SSR
    
    User->>API: 提问 (question)
    API->>Service: run_task()
    
    alt 未指定数据源
        Service->>LLM: 数据源选择
        LLM-->>Service: 选中数据源
    end
    
    Service->>Service: 构建 Prompt (表结构+术语+历史)
    Service->>LLM: 生成 SQL (stream)
    
    loop 流式响应
        LLM-->>Service: SQL 片段
        Service-->>API: SSE 推送
        API-->>User: 实时显示
    end
    
    Service->>Service: 解析 SQL
    Service->>Service: 注入权限过滤
    Service->>DB: 执行 SQL
    DB-->>Service: 查询结果
    
    Service->>LLM: 生成图表配置
    LLM-->>Service: G2 配置 JSON
    
    opt 需要服务端渲染
        Service->>G2: 渲染图表
        G2-->>Service: 图片 URL
    end
    
    Service-->>API: 完整响应
    API-->>User: 展示结果
```

### 2.3 核心方法说明

| 方法 | 职责 | 输入 | 输出 |
| :--- | :--- | :--- | :--- |
| `run_task()` | 主流程入口 | ChatQuestion | Generator (SSE) |
| `generate_sql()` | SQL 生成 | Prompt 消息 | SQL 字符串 |
| `execute_sql()` | SQL 执行 | SQL + 数据源 | DataFrame |
| `generate_chart()` | 图表生成 | 数据 + 问题 | G2 配置 |
| `generate_filter()` | 权限过滤 | SQL + 用户 | 改写后 SQL |

---

## 3. Datasource 模块（数据源管理）

### 3.1 类结构

```mermaid
classDiagram
    class CoreDatasource {
        +int id
        +int oid
        +str name
        +str type
        +str connection_info
        +int status
    }
    
    class CoreTable {
        +int id
        +int ds_id
        +str name
        +str description
        +Vector embedding
    }
    
    class CoreField {
        +int id
        +int table_id
        +str name
        +str type
        +str description
    }
    
    class DBConnector {
        <<interface>>
        +connect()
        +execute(sql)
        +get_tables()
        +get_fields()
    }
    
    class PostgresConnector {
        +connect()
        +execute(sql)
    }
    
    class MySQLConnector {
        +connect()
        +execute(sql)
    }
    
    CoreDatasource "1" --> "*" CoreTable
    CoreTable "1" --> "*" CoreField
    DBConnector <|-- PostgresConnector
    DBConnector <|-- MySQLConnector
```

### 3.2 数据源连接流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant API as Datasource API
    participant CRUD as Datasource CRUD
    participant Connector as DB Connector
    participant ExtDB as 外部数据库
    
    User->>API: 创建数据源
    API->>CRUD: 保存配置
    CRUD->>Connector: 测试连接
    Connector->>ExtDB: ping
    ExtDB-->>Connector: success
    Connector-->>CRUD: 连接成功
    
    CRUD->>Connector: 同步元数据
    Connector->>ExtDB: 获取表列表
    ExtDB-->>Connector: 表列表
    Connector->>ExtDB: 获取字段信息
    ExtDB-->>Connector: 字段信息
    
    CRUD->>CRUD: 保存表和字段
    CRUD->>CRUD: 计算表结构向量
    CRUD-->>API: 创建成功
    API-->>User: 返回数据源 ID
```

---

## 4. AI Model 模块

### 4.1 工厂模式

```mermaid
classDiagram
    class LLMFactory {
        -Dict _llm_types
        +create_llm(config) BaseLLM
        +register_llm(type, class)
    }
    
    class BaseLLM {
        <<abstract>>
        +LLMConfig config
        +BaseChatModel llm
        #_init_llm() BaseChatModel
    }
    
    class OpenAILLM {
        #_init_llm() ChatOpenAI
    }
    
    class AzureLLM {
        #_init_llm() AzureChatOpenAI
    }
    
    class vLLMLLM {
        #_init_llm() VLLMOpenAI
    }
    
    LLMFactory --> BaseLLM
    BaseLLM <|-- OpenAILLM
    BaseLLM <|-- AzureLLM
    BaseLLM <|-- vLLMLLM
```

### 4.2 Embedding 缓存模式

```mermaid
classDiagram
    class EmbeddingModelCache {
        -Dict _embedding_model
        -Lock lock
        +get_model(key, config) Embeddings
    }
    
    class EmbeddingModelInfo {
        +str folder
        +str name
    }
    
    class HuggingFaceEmbeddings {
        +embed_documents(texts)
        +embed_query(text)
    }
    
    EmbeddingModelCache --> EmbeddingModelInfo
    EmbeddingModelCache --> HuggingFaceEmbeddings
```

---

## 5. System 模块

### 5.1 认证流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant API as Login API
    participant Auth as 认证服务
    participant DB as 数据库
    
    User->>API: 登录请求 (用户名, 密码)
    API->>DB: 查询用户
    DB-->>API: 用户信息
    
    API->>Auth: 验证密码
    Auth-->>API: 验证结果
    
    alt 验证成功
        API->>Auth: 生成 JWT
        Auth-->>API: Token
        API-->>User: 返回 Token
    else 验证失败
        API-->>User: 401 错误
    end
```

### 5.2 请求处理流程

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant CORS as CORS 中间件
    participant Token as Token 中间件
    participant Context as 上下文中间件
    participant Router as 路由处理

    Client->>CORS: HTTP 请求
    CORS->>Token: 检查 Token
    
    alt Token 有效
        Token->>Token: 解析用户信息
        Token->>Context: 注入上下文
        Context->>Router: 处理请求
        Router-->>Client: 响应
    else Token 无效
        Token-->>Client: 401 Unauthorized
    end
```

---

## 6. 关键设计决策

### 6.1 为什么不使用 LangChain Agent？

| 维度 | LangChain Agent | SQLBot LLMService |
| :--- | :--- | :--- |
| **确定性** | 低（自主决策） | 高（固定流程） |
| **可控性** | 低 | 高 |
| **调试难度** | 高 | 低 |
| **SQL 安全** | 难以保证 | 可精确控制 |

### 6.2 为什么使用 SSE 而非 WebSocket？

| 维度 | SSE | WebSocket |
| :--- | :--- | :--- |
| **协议** | HTTP | 独立协议 |
| **复杂度** | 低 | 高 |
| **代理兼容** | 好 | 需特殊配置 |
| **双向通信** | 否 | 是 |
| **适用场景** | 服务端推送 | 双向实时 |

SQLBot 只需要服务端向客户端推送 AI 响应，SSE 是更简单的选择。

### 6.3 为什么使用 pgvector 而非专用向量数据库？

1. **简化架构**：无需额外部署向量数据库
2. **事务一致性**：向量与业务数据在同一事务中
3. **查询融合**：可在 SQL 中直接进行向量搜索
4. **性能足够**：当前数据规模下性能满足需求

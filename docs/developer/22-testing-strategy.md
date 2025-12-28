# 测试策略

## 1. 测试金字塔

```mermaid
graph TB
    subgraph 测试层次
        E2E[E2E 测试<br/>10%]
        Integration[集成测试<br/>30%]
        Unit[单元测试<br/>60%]
    end
    
    E2E --> Integration
    Integration --> Unit
```

| 层次 | 覆盖范围 | 执行速度 | 维护成本 |
| :--- | :--- | :--- | :--- |
| **单元测试** | 单个函数/类 | 快 | 低 |
| **集成测试** | 模块间交互 | 中 | 中 |
| **E2E 测试** | 完整用户流程 | 慢 | 高 |

---

## 2. 后端测试

### 2.1 测试框架

| 工具 | 用途 |
| :--- | :--- |
| **pytest** | 测试框架 |
| **pytest-asyncio** | 异步测试 |
| **httpx** | API 测试客户端 |
| **factory-boy** | 测试数据工厂 |
| **pytest-cov** | 覆盖率报告 |

### 2.2 目录结构

```
backend/tests/
├── conftest.py           # 全局 fixture
├── unit/                  # 单元测试
│   ├── test_llm_service.py
│   ├── test_embedding.py
│   └── test_utils.py
├── integration/           # 集成测试
│   ├── test_chat_api.py
│   ├── test_datasource_api.py
│   └── test_user_api.py
└── e2e/                   # 端到端测试
    └── test_full_flow.py
```

### 2.3 单元测试示例

```python
# tests/unit/test_llm_service.py

import pytest
from unittest.mock import Mock, patch
from apps.chat.task.llm import LLMService

class TestLLMService:
    
    @pytest.fixture
    def mock_llm(self):
        return Mock()
    
    def test_check_sql_valid(self):
        """测试有效 SQL 解析"""
        result = '{"success": true, "sql": "SELECT * FROM users"}'
        sql, tables = LLMService.check_sql(result)
        assert sql == "SELECT * FROM users"
    
    def test_check_sql_invalid(self):
        """测试无效响应处理"""
        result = 'invalid json'
        with pytest.raises(SingleMessageError):
            LLMService.check_sql(result)
    
    @patch('apps.ai_model.embedding.EmbeddingModelCache.get_model')
    def test_embedding_similarity(self, mock_get_model):
        """测试向量相似度计算"""
        mock_model = Mock()
        mock_model.embed_query.return_value = [0.1, 0.2, 0.3]
        mock_get_model.return_value = mock_model
        
        # 测试逻辑...
```

### 2.4 集成测试示例

```python
# tests/integration/test_chat_api.py

import pytest
from httpx import AsyncClient
from main import app

@pytest.fixture
async def client():
    async with AsyncClient(app=app, base_url="http://test") as client:
        yield client

@pytest.fixture
async def auth_token(client):
    response = await client.post("/api/v1/login", json={
        "username": "admin",
        "password": "SQLBot@123456"
    })
    return response.json()["data"]["token"]

class TestChatAPI:
    
    @pytest.mark.asyncio
    async def test_get_chat_list(self, client, auth_token):
        """测试获取对话列表"""
        response = await client.get(
            "/api/v1/chat/list",
            headers={"X-SQLBOT-TOKEN": auth_token}
        )
        assert response.status_code == 200
        assert "data" in response.json()
    
    @pytest.mark.asyncio
    async def test_unauthorized_access(self, client):
        """测试未授权访问"""
        response = await client.get("/api/v1/chat/list")
        assert response.status_code == 401
```

### 2.5 测试数据库配置

```python
# tests/conftest.py

import pytest
from sqlalchemy import create_engine
from sqlmodel import Session, SQLModel

TEST_DATABASE_URL = "postgresql://test:test@localhost:5432/sqlbot_test"

@pytest.fixture(scope="session")
def engine():
    engine = create_engine(TEST_DATABASE_URL)
    SQLModel.metadata.create_all(engine)
    yield engine
    SQLModel.metadata.drop_all(engine)

@pytest.fixture
def session(engine):
    with Session(engine) as session:
        yield session
        session.rollback()
```

### 2.6 运行测试

```bash
cd backend

# 运行所有测试
uv run pytest

# 运行单元测试
uv run pytest tests/unit/

# 运行特定测试
uv run pytest tests/unit/test_llm_service.py::TestLLMService::test_check_sql_valid

# 生成覆盖率报告
uv run pytest --cov=apps --cov-report=html
```

---

## 3. 前端测试

### 3.1 测试框架

| 工具 | 用途 |
| :--- | :--- |
| **Vitest** | 单元测试 |
| **Vue Test Utils** | 组件测试 |
| **Playwright** | E2E 测试 |

### 3.2 目录结构

```
frontend/tests/
├── unit/                  # 单元测试
│   ├── utils.spec.ts
│   └── stores.spec.ts
├── components/            # 组件测试
│   ├── ChatMessage.spec.ts
│   └── DataTable.spec.ts
└── e2e/                   # E2E 测试
    ├── login.spec.ts
    └── chat.spec.ts
```

### 3.3 组件测试示例

```typescript
// tests/components/ChatMessage.spec.ts

import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import ChatMessage from '@/components/ChatMessage.vue'

describe('ChatMessage', () => {
  it('renders user message correctly', () => {
    const wrapper = mount(ChatMessage, {
      props: {
        message: {
          role: 'user',
          content: 'Hello'
        }
      }
    })
    
    expect(wrapper.text()).toContain('Hello')
    expect(wrapper.classes()).toContain('user-message')
  })

  it('renders AI message with SQL', () => {
    const wrapper = mount(ChatMessage, {
      props: {
        message: {
          role: 'assistant',
          content: 'SELECT * FROM users',
          type: 'sql'
        }
      }
    })
    
    expect(wrapper.find('.sql-block').exists()).toBe(true)
  })
})
```

### 3.4 E2E 测试示例

```typescript
// tests/e2e/login.spec.ts

import { test, expect } from '@playwright/test'

test.describe('Login', () => {
  test('successful login', async ({ page }) => {
    await page.goto('/login')
    
    await page.fill('input[name="username"]', 'admin')
    await page.fill('input[name="password"]', 'SQLBot@123456')
    await page.click('button[type="submit"]')
    
    await expect(page).toHaveURL('/chat')
    await expect(page.locator('.user-info')).toContainText('admin')
  })

  test('failed login shows error', async ({ page }) => {
    await page.goto('/login')
    
    await page.fill('input[name="username"]', 'admin')
    await page.fill('input[name="password"]', 'wrongpassword')
    await page.click('button[type="submit"]')
    
    await expect(page.locator('.error-message')).toBeVisible()
  })
})
```

### 3.5 运行前端测试

```bash
cd frontend

# 单元测试
pnpm test:unit

# E2E 测试
pnpm test:e2e

# 生成覆盖率报告
pnpm test:coverage
```

---

## 4. 测试用例设计

### 4.1 智能问答测试用例

| 用例 ID | 场景 | 输入 | 预期结果 |
| :--- | :--- | :--- | :--- |
| TC-001 | 简单查询 | "查询用户数量" | 返回 COUNT SQL |
| TC-002 | 时间范围 | "上个月销售额" | 包含时间条件 |
| TC-003 | 多表关联 | "用户的订单" | 包含 JOIN |
| TC-004 | 聚合查询 | "按地区统计" | 包含 GROUP BY |
| TC-005 | 无效问题 | "今天天气" | 拒绝回答 |

### 4.2 数据源测试用例

| 用例 ID | 场景 | 操作 | 预期结果 |
| :--- | :--- | :--- | :--- |
| TC-101 | 创建数据源 | 正确配置 | 成功创建 |
| TC-102 | 连接测试 | 正确配置 | 连接成功 |
| TC-103 | 连接测试 | 错误密码 | 错误提示 |
| TC-104 | 同步表结构 | 有效数据源 | 表已同步 |
| TC-105 | 删除数据源 | 有关联对话 | 提示确认 |

### 4.3 权限测试用例

| 用例 ID | 场景 | 用户角色 | 预期结果 |
| :--- | :--- | :--- | :--- |
| TC-201 | 访问管理页面 | 普通用户 | 403 禁止 |
| TC-202 | 访问其他空间 | 普通用户 | 无数据 |
| TC-203 | 行级权限 | 受限用户 | 数据已过滤 |
| TC-204 | Token 过期 | 任何用户 | 401 未授权 |

---

## 5. 持续集成

### 5.1 GitHub Actions

```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  backend-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: pgvector/pgvector:pg16
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: sqlbot_test
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4
      
      - name: Install uv
        uses: astral-sh/setup-uv@v4
      
      - name: Run tests
        working-directory: backend
        run: |
          uv sync
          uv run pytest --cov=apps --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v4

  frontend-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: pnpm/action-setup@v2
        with:
          version: 8
      
      - name: Install dependencies
        working-directory: frontend
        run: pnpm install
      
      - name: Run tests
        working-directory: frontend
        run: pnpm test:unit
```

---

## 6. 测试覆盖率目标

| 模块 | 目标覆盖率 |
| :--- | :---: |
| **核心业务逻辑** | > 80% |
| **API 接口** | > 70% |
| **工具函数** | > 90% |
| **前端组件** | > 60% |

---

## 7. 测试检查清单

- [ ] 核心功能单元测试
- [ ] API 集成测试
- [ ] 关键用户流程 E2E 测试
- [ ] 权限边界测试
- [ ] 错误处理测试
- [ ] CI 流水线配置
- [ ] 覆盖率报告生成

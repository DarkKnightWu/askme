# 4.3 数据库迁移指南

## 1. 概述

SQLBot 使用 **Alembic** 进行数据库版本管理，确保数据库 Schema 与代码保持同步。

---

## 2. 基础配置

### 2.1 配置文件

```ini
# alembic.ini
[alembic]
script_location = alembic
prepend_sys_path = .
```

### 2.2 环境配置

```python
# alembic/env.py
from common.core.config import settings
from common.core.db import engine

# 使用项目配置的数据库连接
config.set_main_option("sqlalchemy.url", str(settings.SQLALCHEMY_DATABASE_URI))
```

---

## 3. 常用命令

### 3.1 执行迁移

```bash
cd backend

# 升级到最新版本
uv run alembic upgrade head

# 升级到指定版本
uv run alembic upgrade <revision_id>

# 升级一个版本
uv run alembic upgrade +1
```

### 3.2 回滚迁移

```bash
# 回滚一个版本
uv run alembic downgrade -1

# 回滚到指定版本
uv run alembic downgrade <revision_id>

# 回滚到初始状态（危险！）
uv run alembic downgrade base
```

### 3.3 查看状态

```bash
# 查看当前版本
uv run alembic current

# 查看迁移历史
uv run alembic history

# 查看详细历史
uv run alembic history --verbose
```

### 3.4 创建迁移

```bash
# 自动生成迁移（根据模型变化）
uv run alembic revision --autogenerate -m "add_new_table"

# 创建空白迁移
uv run alembic revision -m "custom_migration"
```

---

## 4. 迁移文件结构

### 4.1 文件命名

```
alembic/versions/
├── 001_ddl.py
├── 002_ddl_autogenerate.py
├── 003_add_datasource.py
...
└── 057_update_sys_log.py
```

### 4.2 文件内容

```python
"""add_new_feature

Revision ID: xxx123
Revises: yyy456
Create Date: 2025-01-01 12:00:00.000000
"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa

# 版本标识
revision: str = 'xxx123'
down_revision: Union[str, None] = 'yyy456'
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    """升级操作"""
    op.create_table(
        'new_feature',
        sa.Column('id', sa.BigInteger(), primary_key=True),
        sa.Column('name', sa.String(255), nullable=False),
    )


def downgrade() -> None:
    """回滚操作"""
    op.drop_table('new_feature')
```

---

## 5. 常见操作示例

### 5.1 创建表

```python
def upgrade() -> None:
    op.create_table(
        'my_table',
        sa.Column('id', sa.BigInteger(), primary_key=True),
        sa.Column('name', sa.String(255), nullable=False),
        sa.Column('description', sa.Text(), nullable=True),
        sa.Column('created_at', sa.BigInteger(), default=0),
        sa.Column('oid', sa.BigInteger(), nullable=True),
    )
    
    # 创建索引
    op.create_index('idx_my_table_oid', 'my_table', ['oid'])

def downgrade() -> None:
    op.drop_index('idx_my_table_oid', 'my_table')
    op.drop_table('my_table')
```

### 5.2 修改表

```python
def upgrade() -> None:
    # 添加列
    op.add_column('my_table', sa.Column('new_column', sa.String(100)))
    
    # 修改列类型
    op.alter_column('my_table', 'name', type_=sa.String(500))
    
    # 重命名列
    op.alter_column('my_table', 'old_name', new_column_name='new_name')

def downgrade() -> None:
    op.alter_column('my_table', 'new_name', new_column_name='old_name')
    op.alter_column('my_table', 'name', type_=sa.String(255))
    op.drop_column('my_table', 'new_column')
```

### 5.3 删除表

```python
def upgrade() -> None:
    op.drop_table('deprecated_table')

def downgrade() -> None:
    op.create_table(
        'deprecated_table',
        # ... 原表结构
    )
```

### 5.4 创建向量列 (pgvector)

```python
from pgvector.sqlalchemy import VECTOR

def upgrade() -> None:
    op.add_column('terminology', sa.Column('embedding', VECTOR(), nullable=True))
    
    # 创建向量索引
    op.execute("""
        CREATE INDEX idx_terminology_embedding 
        ON terminology 
        USING ivfflat (embedding vector_cosine_ops) 
        WITH (lists = 100)
    """)

def downgrade() -> None:
    op.execute("DROP INDEX IF EXISTS idx_terminology_embedding")
    op.drop_column('terminology', 'embedding')
```

### 5.5 数据迁移

```python
def upgrade() -> None:
    # 执行原始 SQL
    op.execute("""
        UPDATE my_table 
        SET status = 1 
        WHERE status IS NULL
    """)

def downgrade() -> None:
    # 回滚逻辑
    op.execute("""
        UPDATE my_table 
        SET status = NULL 
        WHERE status = 1
    """)
```

---

## 6. 最佳实践

### 6.1 开发流程

1. **修改模型**：在 `models/*.py` 中修改数据模型
2. **生成迁移**：`uv run alembic revision --autogenerate -m "描述"`
3. **检查迁移**：确认生成的迁移文件正确
4. **执行迁移**：`uv run alembic upgrade head`
5. **提交代码**：将迁移文件加入版本控制

### 6.2 注意事项

> [!WARNING]
> - **永远不要** 修改已提交的迁移文件
> - **永远不要** 删除已执行的迁移文件
> - 生产环境迁移前 **务必备份数据**

### 6.3 命名规范

```
{序号}_{描述}.py

示例：
058_add_user_avatar.py
059_modify_chat_record.py
060_create_api_key.py
```

---

## 7. 故障排查

### 7.1 版本冲突

**症状**：
```
alembic.util.exc.CommandError: Can't locate revision identified by 'xxx'
```

**解决方案**：
```bash
# 查看当前状态
uv run alembic current

# 强制设置版本（谨慎）
uv run alembic stamp head
```

### 7.2 迁移失败

**症状**：
```
sqlalchemy.exc.ProgrammingError: relation "xxx" already exists
```

**解决方案**：
1. 检查是否重复执行迁移
2. 手动修复数据库状态
3. 使用 `alembic stamp` 跳过问题版本

### 7.3 自动生成不准确

**原因**：Alembic 无法检测所有变化

**解决方案**：
1. 手动检查生成的迁移文件
2. 补充缺失的操作
3. 测试升级和降级

---

## 8. 生产环境部署

### 8.1 部署前检查

```bash
# 查看待执行的迁移
uv run alembic history --indicate-current

# 空运行（查看 SQL）
uv run alembic upgrade head --sql > migration.sql
```

### 8.2 部署步骤

1. **备份数据库**
```bash
pg_dump -U postgres sqlbot > backup_$(date +%Y%m%d).sql
```

2. **执行迁移**
```bash
uv run alembic upgrade head
```

3. **验证**
```bash
uv run alembic current
```

### 8.3 回滚方案

```bash
# 如果出现问题
uv run alembic downgrade -1

# 恢复备份（最后手段）
psql -U postgres sqlbot < backup_xxx.sql
```

# 5.3 生产环境部署

## 1. 部署架构

### 1.1 推荐架构

```mermaid
graph TB
    subgraph 负载均衡
        LB[Nginx / 云 LB]
    end
    
    subgraph 应用层
        App1[SQLBot Node 1]
        App2[SQLBot Node 2]
    end
    
    subgraph 数据层
        PG[(PostgreSQL 主)]
        PG_S[(PostgreSQL 从)]
        Redis[(Redis)]
    end
    
    subgraph 存储层
        NFS[共享存储 NFS/OSS]
    end
    
    User[用户] --> LB
    LB --> App1
    LB --> App2
    App1 --> PG
    App2 --> PG
    PG --> PG_S
    App1 --> Redis
    App2 --> Redis
    App1 --> NFS
    App2 --> NFS
```

### 1.2 资源规划

| 组件 | 最小配置 | 推荐配置 | 说明 |
| :--- | :--- | :--- | :--- |
| **应用节点** | 2C4G | 4C8G | 启用 Embedding 需更多内存 |
| **PostgreSQL** | 2C4G | 4C8G | 含 pgvector |
| **Redis** | 1C1G | 2C2G | 可选 |
| **存储** | 50GB | 200GB | 含日志、上传文件 |

---

## 2. 基础设施准备

### 2.1 PostgreSQL 集群

```bash
# 主库配置 (postgresql.conf)
wal_level = replica
max_wal_senders = 3
synchronous_commit = on

# 从库配置
hot_standby = on
```

### 2.2 Redis 集群（可选）

```bash
# 配置 Redis Sentinel 或 Cluster
redis-server --port 6379 --cluster-enabled yes
```

### 2.3 共享存储

上传的文件需要在多节点间共享：

```bash
# NFS 挂载
mount -t nfs storage-server:/sqlbot /opt/sqlbot/data

# 或使用云对象存储 (OSS/S3)
```

---

## 3. 应用部署

### 3.1 Docker Compose（多节点）

```yaml
# docker-compose.prod.yaml
version: '3.8'

services:
  sqlbot:
    image: dataease/sqlbot:latest
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '4'
          memory: 8G
    environment:
      # 外部数据库
      SQLBOT_DB_URL: postgresql+psycopg://user:pass@pg-cluster:5432/sqlbot
      
      # Redis 缓存
      CACHE_TYPE: redis
      CACHE_REDIS_URL: redis://redis-cluster:6379/0
      
      # 日志级别
      LOG_LEVEL: WARNING
      
      # 安全配置
      SECRET_KEY: ${SECRET_KEY}
    volumes:
      - /mnt/nfs/sqlbot/data:/opt/sqlbot/data
      - /mnt/nfs/sqlbot/images:/opt/sqlbot/images
      - ./logs:/opt/sqlbot/app/logs
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/api/v1/settings/basic"]
      interval: 30s
      timeout: 10s
      retries: 3

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./certs:/etc/nginx/certs
    depends_on:
      - sqlbot
```

### 3.2 Kubernetes 部署

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sqlbot
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sqlbot
  template:
    metadata:
      labels:
        app: sqlbot
    spec:
      containers:
      - name: sqlbot
        image: dataease/sqlbot:latest
        resources:
          requests:
            memory: "4Gi"
            cpu: "2"
          limits:
            memory: "8Gi"
            cpu: "4"
        env:
        - name: SQLBOT_DB_URL
          valueFrom:
            secretKeyRef:
              name: sqlbot-secrets
              key: database-url
        ports:
        - containerPort: 8000
        livenessProbe:
          httpGet:
            path: /api/v1/settings/basic
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        volumeMounts:
        - name: data
          mountPath: /opt/sqlbot/data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: sqlbot-data
---
apiVersion: v1
kind: Service
metadata:
  name: sqlbot
spec:
  ports:
  - port: 8000
    targetPort: 8000
  selector:
    app: sqlbot
```

---

## 4. Nginx 配置

### 4.1 反向代理配置

```nginx
# nginx.conf
upstream sqlbot {
    least_conn;
    server sqlbot1:8000 weight=1;
    server sqlbot2:8000 weight=1;
    keepalive 32;
}

server {
    listen 80;
    server_name sqlbot.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name sqlbot.example.com;
    
    # SSL 配置
    ssl_certificate /etc/nginx/certs/cert.pem;
    ssl_certificate_key /etc/nginx/certs/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256;
    
    # 安全头
    add_header X-Frame-Options SAMEORIGIN;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    
    # 文件大小限制
    client_max_body_size 100M;
    
    location / {
        proxy_pass http://sqlbot;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # SSE 支持
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
    }
    
    # 静态文件缓存
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        proxy_pass http://sqlbot;
        expires 7d;
        add_header Cache-Control "public, immutable";
    }
}
```

---

## 5. 安全加固

### 5.1 环境变量

```bash
# 生产环境必须设置
SECRET_KEY=<强随机字符串>
DEFAULT_PWD=<复杂密码>

# 限制 CORS
BACKEND_CORS_ORIGINS=https://sqlbot.example.com

# 日志级别
LOG_LEVEL=WARNING
```

### 5.2 防火墙规则

```bash
# 仅开放必要端口
ufw allow 80/tcp
ufw allow 443/tcp
ufw deny 8000/tcp    # 禁止直接访问应用端口
ufw deny 5432/tcp    # 数据库端口内网访问
```

### 5.3 数据库安全

```sql
-- 创建专用用户
CREATE USER sqlbot WITH PASSWORD 'strong_password';
GRANT CONNECT ON DATABASE sqlbot TO sqlbot;
GRANT USAGE ON SCHEMA public TO sqlbot;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO sqlbot;

-- 只读用户（用于业务数据源连接）
CREATE USER sqlbot_readonly WITH PASSWORD 'readonly_password';
GRANT CONNECT ON DATABASE business_db TO sqlbot_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO sqlbot_readonly;
```

---

## 6. 监控配置

### 6.1 健康检查

```bash
# 应用健康检查
curl -f http://localhost:8000/api/v1/settings/basic

# 数据库健康检查
pg_isready -h localhost -p 5432 -U postgres
```

### 6.2 日志收集

```yaml
# Filebeat 配置
filebeat.inputs:
  - type: log
    paths:
      - /opt/sqlbot/app/logs/*.log
    json.keys_under_root: true

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "sqlbot-%{+yyyy.MM.dd}"
```

### 6.3 指标监控

```yaml
# Prometheus 抓取配置
scrape_configs:
  - job_name: 'sqlbot'
    static_configs:
      - targets: ['sqlbot1:8000', 'sqlbot2:8000']
    metrics_path: /metrics  # 需启用指标端点
```

---

## 7. 备份策略

### 7.1 数据库备份

```bash
# 每日全量备份
0 2 * * * pg_dump -U postgres sqlbot | gzip > /backup/sqlbot_$(date +\%Y\%m\%d).sql.gz

# 保留 30 天
find /backup -name "sqlbot_*.sql.gz" -mtime +30 -delete
```

### 7.2 文件备份

```bash
# 同步到备份存储
rsync -avz /opt/sqlbot/data/ backup-server:/backup/sqlbot/data/
```

---

## 8. 部署检查清单

### 8.1 部署前

- [ ] 数据库已初始化并启用 pgvector
- [ ] Redis 已配置（如需）
- [ ] 共享存储已挂载
- [ ] SSL 证书已准备
- [ ] 环境变量已配置
- [ ] 防火墙规则已设置

### 8.2 部署后

- [ ] 应用可正常访问
- [ ] HTTPS 正常工作
- [ ] 健康检查通过
- [ ] 日志正常输出
- [ ] 备份任务已配置
- [ ] 监控已接入
- [ ] 默认密码已修改

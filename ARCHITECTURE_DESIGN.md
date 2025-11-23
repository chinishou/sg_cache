# ShotGrid Cache 架構設計 - 三種情境方案

## 概述

本文檔定義了三種不同規模和需求下的 ShotGrid 緩存中間層架構設計。所有方案均支援 Linux 和 Windows 平台。

## 設計目標

1. **效能提升**：避免直接訪問 ShotGrid 的加密握手開銷
2. **快速響應**：本地機器使用標準 HTTP 通信
3. **智能緩存**：訂閱 ShotGrid 事件進行緩存失效
4. **可擴展性**：支援未來資料庫遷移
5. **跨平台**：Linux 和 Windows 環境均可運行

---

## 情境一：輕量級單實例架構（Small Team / Low Load）

### 適用場景
- 小型團隊（10-50 人）
- 日訪問量 < 10,000 次
- 預算有限，無需額外基礎設施
- 可接受服務重啟時緩存重建

### 架構圖
```
┌─────────────┐
│   Clients   │ (Artists, TDs, Pipeline Tools)
│ Linux/Win   │
└──────┬──────┘
       │ HTTP
       ▼
┌─────────────────────────────────┐
│   FastAPI Service (Single)      │
│  ┌──────────────────────────┐   │
│  │   In-Memory Cache        │   │
│  │   (Python dict + LRU)    │   │
│  └──────────────────────────┘   │
│  ┌──────────────────────────┐   │
│  │  ShotGrid Event Listener │   │
│  └──────────────────────────┘   │
└────────────┬────────────────────┘
             │ HTTPS + Auth
             ▼
      ┌──────────────┐
      │  ShotGrid    │
      │   Server     │
      └──────────────┘
```

### 技術棧
- **Web 框架**：FastAPI (Python 3.9+)
- **緩存存儲**：內建 Python dict + cachetools (LRU)
- **事件監聽**：ShotGrid EventStream API
- **部署方式**：
  - Linux: systemd service
  - Windows: NSSM (Non-Sucking Service Manager) 或 Windows Service
- **配置管理**：YAML 配置文件

### 核心組件

#### 1. 緩存管理器
```python
from cachetools import TTLCache
import threading

class CacheManager:
    def __init__(self, maxsize=10000, ttl=3600):
        self.cache = TTLCache(maxsize=maxsize, ttl=ttl)
        self.lock = threading.RLock()

    def get(self, key):
        with self.lock:
            return self.cache.get(key)

    def set(self, key, value):
        with self.lock:
            self.cache[key] = value

    def invalidate(self, key):
        with self.lock:
            self.cache.pop(key, None)
```

#### 2. 查詢處理策略
- **頂層對象**（Projects, Shots, Assets）：全量緩存，按 ID 查詢
- **複雜查詢**：將查詢作為不透明鍵（opaque key），緩存 ShotGrid 響應
- **緩存鍵格式**：`{entity_type}:{operation}:{hash(query_params)}`

#### 3. 事件處理
```python
# 偽代碼
def handle_shotgrid_event(event):
    entity_type = event['meta']['entity_type']
    entity_id = event['meta']['entity_id']

    # 失效相關緩存
    cache.invalidate(f"{entity_type}:{entity_id}")
    cache.invalidate_pattern(f"{entity_type}:query:*")
```

### 部署配置

#### Linux (systemd)
```ini
# /etc/systemd/system/sg-cache.service
[Unit]
Description=ShotGrid Cache Service
After=network.target

[Service]
Type=simple
User=pipeline
WorkingDirectory=/opt/sg_cache
Environment="PATH=/opt/sg_cache/venv/bin"
ExecStart=/opt/sg_cache/venv/bin/python -m uvicorn main:app --host 0.0.0.0 --port 8000
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

#### Windows (NSSM)
```powershell
# 安裝服務
nssm install ShotGridCache "C:\sg_cache\venv\Scripts\python.exe" "-m uvicorn main:app --host 0.0.0.0 --port 8000"
nssm set ShotGridCache AppDirectory "C:\sg_cache"
nssm set ShotGridCache Start SERVICE_AUTO_START
nssm start ShotGridCache
```

### 優點
- ✅ 零外部依賴
- ✅ 部署簡單
- ✅ 維護成本低
- ✅ 響應速度快（純內存）

### 缺點
- ❌ 單點故障
- ❌ 無法橫向擴展
- ❌ 重啟後緩存丟失
- ❌ 內存限制緩存容量

---

## 情境二：中等規模多實例架構（Medium Team / Moderate Load）

### 適用場景
- 中型團隊（50-200 人）
- 日訪問量 10,000 - 100,000 次
- 需要高可用性
- 有基礎設施團隊支持

### 架構圖
```
┌─────────────┐
│   Clients   │ (Artists, TDs, Pipeline Tools)
│ Linux/Win   │
└──────┬──────┘
       │ HTTP
       ▼
┌─────────────────┐
│  Load Balancer  │ (Nginx / HAProxy)
│  (Round Robin)  │
└────────┬────────┘
         │
    ┌────┴────┬────────┬─────────┐
    ▼         ▼        ▼         ▼
┌────────┐ ┌────────┐ ┌────────┐ ...
│FastAPI │ │FastAPI │ │FastAPI │
│Instance│ │Instance│ │Instance│
│   #1   │ │   #2   │ │   #3   │
└───┬────┘ └───┬────┘ └───┬────┘
    │          │          │
    └──────────┼──────────┘
               │
               ▼
        ┌─────────────┐
        │    Redis    │ (Shared Cache)
        │   Cluster   │
        │ (Master +   │
        │  Replicas)  │
        └──────┬──────┘
               │
               │ HTTPS + Auth
               ▼
        ┌──────────────┐
        │  ShotGrid    │
        │   Server     │
        └──────────────┘
```

### 技術棧
- **Web 框架**：FastAPI (Python 3.9+)
- **緩存存儲**：Redis 7.x
- **負載均衡**：Nginx 或 HAProxy
- **事件監聽**：ShotGrid EventStream API
- **部署方式**：
  - Linux: Docker Compose 或 systemd
  - Windows: Docker Desktop 或 NSSM
- **監控**：Prometheus + Grafana

### 核心組件

#### 1. Redis 緩存策略
```python
import redis
import hashlib
import json

class RedisCache:
    def __init__(self, host='localhost', port=6379, db=0):
        self.client = redis.Redis(
            host=host,
            port=port,
            db=db,
            decode_responses=True
        )

    def get_entity(self, entity_type, entity_id):
        key = f"entity:{entity_type}:{entity_id}"
        data = self.client.get(key)
        return json.loads(data) if data else None

    def set_entity(self, entity_type, entity_id, data, ttl=3600):
        key = f"entity:{entity_type}:{entity_id}"
        self.client.setex(key, ttl, json.dumps(data))

    def get_query(self, entity_type, filters, fields):
        query_key = self._hash_query(entity_type, filters, fields)
        key = f"query:{entity_type}:{query_key}"
        data = self.client.get(key)
        return json.loads(data) if data else None

    def set_query(self, entity_type, filters, fields, data, ttl=1800):
        query_key = self._hash_query(entity_type, filters, fields)
        key = f"query:{entity_type}:{query_key}"
        self.client.setex(key, ttl, json.dumps(data))

    def invalidate_entity(self, entity_type, entity_id):
        # 失效單個實體
        self.client.delete(f"entity:{entity_type}:{entity_id}")
        # 失效相關查詢（使用 SCAN + DEL）
        cursor = 0
        while True:
            cursor, keys = self.client.scan(
                cursor,
                match=f"query:{entity_type}:*",
                count=100
            )
            if keys:
                self.client.delete(*keys)
            if cursor == 0:
                break

    @staticmethod
    def _hash_query(entity_type, filters, fields):
        query_str = f"{entity_type}:{json.dumps(filters, sort_keys=True)}:{json.dumps(fields, sort_keys=True)}"
        return hashlib.sha256(query_str.encode()).hexdigest()[:16]
```

#### 2. FastAPI 端點示例
```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import shotgun_api3

app = FastAPI()
cache = RedisCache()
sg = shotgun_api3.Shotgun(SG_URL, script_name=SG_SCRIPT, api_key=SG_KEY)

class QueryRequest(BaseModel):
    entity_type: str
    filters: list
    fields: list

@app.get("/entity/{entity_type}/{entity_id}")
async def get_entity(entity_type: str, entity_id: int):
    # 檢查緩存
    cached = cache.get_entity(entity_type, entity_id)
    if cached:
        return {"data": cached, "source": "cache"}

    # 查詢 ShotGrid
    data = sg.find_one(entity_type, [['id', 'is', entity_id]])
    if not data:
        raise HTTPException(status_code=404, detail="Entity not found")

    # 緩存結果
    cache.set_entity(entity_type, entity_id, data)
    return {"data": data, "source": "shotgrid"}

@app.post("/query")
async def query_entities(req: QueryRequest):
    # 檢查緩存
    cached = cache.get_query(req.entity_type, req.filters, req.fields)
    if cached:
        return {"data": cached, "source": "cache"}

    # 查詢 ShotGrid
    data = sg.find(req.entity_type, req.filters, req.fields)

    # 緩存結果
    cache.set_query(req.entity_type, req.filters, req.fields, data)
    return {"data": data, "source": "shotgrid"}
```

#### 3. 事件監聽服務（獨立進程）
```python
import threading
from shotgun_api3.lib.sgtimezone import SgTimezone

class EventListener:
    def __init__(self, sg_connection, cache):
        self.sg = sg_connection
        self.cache = cache
        self.running = False

    def start(self):
        self.running = True
        thread = threading.Thread(target=self._listen)
        thread.daemon = True
        thread.start()

    def _listen(self):
        last_id = self._get_last_event_id()

        while self.running:
            events = self.sg.find(
                'EventLogEntry',
                [['id', 'greater_than', last_id]],
                ['id', 'event_type', 'entity', 'meta'],
                order=[{'field_name': 'id', 'direction': 'asc'}],
                limit=100
            )

            for event in events:
                self._process_event(event)
                last_id = event['id']

            time.sleep(5)  # 每 5 秒輪詢

    def _process_event(self, event):
        if event['event_type'] in ['Shotgun_Asset_Change',
                                     'Shotgun_Shot_Change',
                                     'Shotgun_Project_Change']:
            entity_type = event['entity']['type']
            entity_id = event['entity']['id']
            self.cache.invalidate_entity(entity_type, entity_id)

    def _get_last_event_id(self):
        # 從配置或 Redis 獲取
        return int(self.cache.client.get('last_event_id') or 0)
```

### 部署配置

#### Docker Compose (Linux/Windows)
```yaml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes
    restart: always

  sg-cache-api-1:
    build: .
    ports:
      - "8001:8000"
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - SG_URL=${SG_URL}
      - SG_SCRIPT=${SG_SCRIPT}
      - SG_KEY=${SG_KEY}
    depends_on:
      - redis
    restart: always

  sg-cache-api-2:
    build: .
    ports:
      - "8002:8000"
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - SG_URL=${SG_URL}
      - SG_SCRIPT=${SG_SCRIPT}
      - SG_KEY=${SG_KEY}
    depends_on:
      - redis
    restart: always

  sg-cache-api-3:
    build: .
    ports:
      - "8003:8000"
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - SG_URL=${SG_URL}
      - SG_SCRIPT=${SG_SCRIPT}
      - SG_KEY=${SG_KEY}
    depends_on:
      - redis
    restart: always

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - sg-cache-api-1
      - sg-cache-api-2
      - sg-cache-api-3
    restart: always

  event-listener:
    build: .
    command: python event_listener.py
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - SG_URL=${SG_URL}
      - SG_SCRIPT=${SG_SCRIPT}
      - SG_KEY=${SG_KEY}
    depends_on:
      - redis
    restart: always

volumes:
  redis-data:
```

#### Nginx 配置
```nginx
upstream sg_cache {
    least_conn;
    server sg-cache-api-1:8000;
    server sg-cache-api-2:8000;
    server sg-cache-api-3:8000;
}

server {
    listen 80;

    location / {
        proxy_pass http://sg_cache;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # 超時設置
        proxy_connect_timeout 10s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }

    location /health {
        access_log off;
        return 200 "OK\n";
    }
}
```

### 優點
- ✅ 高可用性（無單點故障）
- ✅ 水平擴展能力
- ✅ 緩存持久化（Redis AOF/RDB）
- ✅ 適合中等規模團隊
- ✅ 容器化部署簡單

### 缺點
- ❌ 需要維護 Redis 集群
- ❌ 架構複雜度增加
- ❌ 資源消耗較高

---

## 情境三：企業級架構（Large Team / High Load + Persistence）

### 適用場景
- 大型團隊（200+ 人）
- 日訪問量 > 100,000 次
- 需要複雜關聯查詢
- 需要數據持久化和審計
- 需要歷史數據分析

### 架構圖
```
┌─────────────┐
│   Clients   │ (Artists, TDs, Pipeline Tools)
│ Linux/Win   │
└──────┬──────┘
       │ HTTP
       ▼
┌─────────────────┐
│  Load Balancer  │ (Nginx / HAProxy)
│   + SSL Term    │
└────────┬────────┘
         │
    ┌────┴────┬────────┬─────────┐
    ▼         ▼        ▼         ▼
┌────────┐ ┌────────┐ ┌────────┐ ...
│FastAPI │ │FastAPI │ │FastAPI │
│Instance│ │Instance│ │Instance│
└───┬────┘ └───┬────┘ └───┬────┘
    │          │          │
    ├──────────┼──────────┤
    │          │          │
    ▼          ▼          ▼
┌─────────────────────────────┐
│     Redis (L1 Cache)        │
│    (Hot Data, TTL: 5min)    │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│  PostgreSQL (L2 Storage)    │
│  ┌─────────────────────┐    │
│  │ Entities Table      │    │
│  │ (JSON + Index)      │    │
│  ├─────────────────────┤    │
│  │ Query Cache Table   │    │
│  ├─────────────────────┤    │
│  │ Event Log Table     │    │
│  │ (Audit Trail)       │    │
│  └─────────────────────┘    │
│  Master + Read Replicas     │
└─────────────┬───────────────┘
              │
              │ HTTPS + Auth
              ▼
       ┌──────────────┐
       │  ShotGrid    │
       │   Server     │
       └──────────────┘
```

### 技術棧
- **Web 框架**：FastAPI (Python 3.9+)
- **L1 緩存**：Redis 7.x (熱數據緩存)
- **L2 存儲**：PostgreSQL 15+ (持久化存儲)
- **負載均衡**：Nginx 或 HAProxy
- **容器編排**：Docker Compose 或 Kubernetes
- **監控**：Prometheus + Grafana + ELK
- **部署方式**：
  - Linux: Kubernetes 或 Docker Swarm
  - Windows: Docker Desktop + Kubernetes

### 數據庫設計

#### PostgreSQL Schema
```sql
-- 實體緩存表
CREATE TABLE entities (
    id BIGSERIAL PRIMARY KEY,
    entity_type VARCHAR(100) NOT NULL,
    entity_id INTEGER NOT NULL,
    data JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(entity_type, entity_id)
);

-- 為 JSONB 查詢創建索引
CREATE INDEX idx_entities_type ON entities(entity_type);
CREATE INDEX idx_entities_data_gin ON entities USING GIN (data);

-- 查詢緩存表
CREATE TABLE query_cache (
    id BIGSERIAL PRIMARY KEY,
    query_hash VARCHAR(64) NOT NULL UNIQUE,
    entity_type VARCHAR(100) NOT NULL,
    filters JSONB NOT NULL,
    fields JSONB NOT NULL,
    result JSONB NOT NULL,
    hit_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    last_accessed TIMESTAMP DEFAULT NOW(),
    ttl INTERVAL DEFAULT '30 minutes'
);

CREATE INDEX idx_query_cache_type ON query_cache(entity_type);
CREATE INDEX idx_query_cache_accessed ON query_cache(last_accessed);

-- 事件日誌表（審計追蹤）
CREATE TABLE event_log (
    id BIGSERIAL PRIMARY KEY,
    sg_event_id INTEGER UNIQUE,
    event_type VARCHAR(100),
    entity_type VARCHAR(100),
    entity_id INTEGER,
    user_name VARCHAR(255),
    meta JSONB,
    processed_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_event_log_entity ON event_log(entity_type, entity_id);
CREATE INDEX idx_event_log_time ON event_log(processed_at);

-- 緩存統計表
CREATE TABLE cache_stats (
    id BIGSERIAL PRIMARY KEY,
    endpoint VARCHAR(255),
    hit_count INTEGER DEFAULT 0,
    miss_count INTEGER DEFAULT 0,
    error_count INTEGER DEFAULT 0,
    avg_response_time_ms FLOAT,
    date DATE DEFAULT CURRENT_DATE,
    UNIQUE(endpoint, date)
);
```

### 核心組件

#### 1. 多層緩存管理器
```python
import redis
import asyncpg
import json
from typing import Optional, Dict, Any

class TieredCacheManager:
    def __init__(self, redis_client: redis.Redis, pg_pool: asyncpg.Pool):
        self.redis = redis_client
        self.pg_pool = pg_pool

    async def get_entity(self, entity_type: str, entity_id: int) -> Optional[Dict[str, Any]]:
        # L1: Redis 緩存
        redis_key = f"entity:{entity_type}:{entity_id}"
        cached = self.redis.get(redis_key)
        if cached:
            return json.loads(cached)

        # L2: PostgreSQL
        async with self.pg_pool.acquire() as conn:
            row = await conn.fetchrow(
                "SELECT data FROM entities WHERE entity_type = $1 AND entity_id = $2",
                entity_type, entity_id
            )

            if row:
                data = row['data']
                # 回寫到 Redis
                self.redis.setex(redis_key, 300, json.dumps(data))
                return data

        return None

    async def set_entity(self, entity_type: str, entity_id: int, data: Dict[str, Any]):
        # 寫入 PostgreSQL
        async with self.pg_pool.acquire() as conn:
            await conn.execute(
                """
                INSERT INTO entities (entity_type, entity_id, data, updated_at)
                VALUES ($1, $2, $3, NOW())
                ON CONFLICT (entity_type, entity_id)
                DO UPDATE SET data = $3, updated_at = NOW()
                """,
                entity_type, entity_id, json.dumps(data)
            )

        # 寫入 Redis
        redis_key = f"entity:{entity_type}:{entity_id}"
        self.redis.setex(redis_key, 300, json.dumps(data))

    async def invalidate_entity(self, entity_type: str, entity_id: int):
        # 從 Redis 刪除
        redis_key = f"entity:{entity_type}:{entity_id}"
        self.redis.delete(redis_key)

        # 從 PostgreSQL 刪除或標記
        async with self.pg_pool.acquire() as conn:
            await conn.execute(
                "DELETE FROM entities WHERE entity_type = $1 AND entity_id = $2",
                entity_type, entity_id
            )

            # 失效相關查詢緩存
            await conn.execute(
                "DELETE FROM query_cache WHERE entity_type = $1",
                entity_type
            )
```

#### 2. 查詢緩存與統計
```python
from datetime import datetime, timedelta
import hashlib

class QueryCacheManager:
    def __init__(self, cache_manager: TieredCacheManager):
        self.cache = cache_manager

    async def get_query(self, entity_type: str, filters: list, fields: list) -> Optional[list]:
        query_hash = self._hash_query(entity_type, filters, fields)

        async with self.cache.pg_pool.acquire() as conn:
            row = await conn.fetchrow(
                """
                SELECT result, last_accessed, ttl
                FROM query_cache
                WHERE query_hash = $1
                """,
                query_hash
            )

            if row:
                # 檢查 TTL
                if datetime.now() - row['last_accessed'] < row['ttl']:
                    # 更新訪問時間和計數
                    await conn.execute(
                        """
                        UPDATE query_cache
                        SET hit_count = hit_count + 1, last_accessed = NOW()
                        WHERE query_hash = $1
                        """,
                        query_hash
                    )
                    return row['result']
                else:
                    # 過期，刪除
                    await conn.execute(
                        "DELETE FROM query_cache WHERE query_hash = $1",
                        query_hash
                    )

        return None

    async def set_query(self, entity_type: str, filters: list, fields: list, result: list):
        query_hash = self._hash_query(entity_type, filters, fields)

        async with self.cache.pg_pool.acquire() as conn:
            await conn.execute(
                """
                INSERT INTO query_cache (query_hash, entity_type, filters, fields, result)
                VALUES ($1, $2, $3, $4, $5)
                ON CONFLICT (query_hash)
                DO UPDATE SET result = $5, last_accessed = NOW(), hit_count = 0
                """,
                query_hash, entity_type, json.dumps(filters), json.dumps(fields), json.dumps(result)
            )

    @staticmethod
    def _hash_query(entity_type: str, filters: list, fields: list) -> str:
        query_str = f"{entity_type}:{json.dumps(filters, sort_keys=True)}:{json.dumps(fields, sort_keys=True)}"
        return hashlib.sha256(query_str.encode()).hexdigest()
```

#### 3. 事件處理與審計
```python
class EnterpriseEventListener:
    def __init__(self, sg, pg_pool: asyncpg.Pool, cache_manager: TieredCacheManager):
        self.sg = sg
        self.pg_pool = pg_pool
        self.cache = cache_manager
        self.running = False

    async def start(self):
        self.running = True
        while self.running:
            await self._process_events()
            await asyncio.sleep(5)

    async def _process_events(self):
        last_id = await self._get_last_event_id()

        events = self.sg.find(
            'EventLogEntry',
            [['id', 'greater_than', last_id]],
            ['id', 'event_type', 'entity', 'meta', 'user'],
            order=[{'field_name': 'id', 'direction': 'asc'}],
            limit=100
        )

        for event in events:
            await self._process_event(event)
            await self._log_event(event)

    async def _process_event(self, event):
        if event['entity']:
            entity_type = event['entity']['type']
            entity_id = event['entity']['id']
            await self.cache.invalidate_entity(entity_type, entity_id)

    async def _log_event(self, event):
        async with self.pg_pool.acquire() as conn:
            await conn.execute(
                """
                INSERT INTO event_log (sg_event_id, event_type, entity_type, entity_id, user_name, meta)
                VALUES ($1, $2, $3, $4, $5, $6)
                ON CONFLICT (sg_event_id) DO NOTHING
                """,
                event['id'],
                event['event_type'],
                event['entity']['type'] if event['entity'] else None,
                event['entity']['id'] if event['entity'] else None,
                event['user']['name'] if event['user'] else None,
                json.dumps(event['meta'])
            )

    async def _get_last_event_id(self) -> int:
        async with self.pg_pool.acquire() as conn:
            row = await conn.fetchrow("SELECT MAX(sg_event_id) as max_id FROM event_log")
            return row['max_id'] or 0
```

### 部署配置

#### Kubernetes Deployment (推薦用於 Linux 生產環境)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sg-cache-config
data:
  SG_URL: "https://your-studio.shotgunstudio.com"
  REDIS_HOST: "redis-service"
  POSTGRES_HOST: "postgres-service"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sg-cache-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: sg-cache-api
  template:
    metadata:
      labels:
        app: sg-cache-api
    spec:
      containers:
      - name: api
        image: sg-cache-api:latest
        ports:
        - containerPort: 8000
        envFrom:
        - configMapRef:
            name: sg-cache-config
        - secretRef:
            name: sg-cache-secrets
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: sg-cache-service
spec:
  selector:
    app: sg-cache-api
  ports:
  - port: 80
    targetPort: 8000
  type: LoadBalancer

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-service
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: sg_cache
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 100Gi

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        command: ["redis-server", "--appendonly", "yes"]
        volumeMounts:
        - name: redis-storage
          mountPath: /data
      volumes:
      - name: redis-storage
        persistentVolumeClaim:
          claimName: redis-pvc
```

#### Docker Compose (適合 Windows 或開發環境)
```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: sg_cache
      POSTGRES_USER: ${PG_USER}
      POSTGRES_PASSWORD: ${PG_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    restart: always
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${PG_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --maxmemory 2gb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    ports:
      - "6379:6379"
    restart: always

  sg-cache-api:
    build: .
    deploy:
      replicas: 3
    environment:
      - REDIS_HOST=redis
      - POSTGRES_HOST=postgres
      - POSTGRES_DB=sg_cache
      - POSTGRES_USER=${PG_USER}
      - POSTGRES_PASSWORD=${PG_PASSWORD}
      - SG_URL=${SG_URL}
      - SG_SCRIPT=${SG_SCRIPT}
      - SG_KEY=${SG_KEY}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    restart: always

  event-listener:
    build: .
    command: python event_listener.py
    environment:
      - REDIS_HOST=redis
      - POSTGRES_HOST=postgres
      - POSTGRES_DB=sg_cache
      - POSTGRES_USER=${PG_USER}
      - POSTGRES_PASSWORD=${PG_PASSWORD}
      - SG_URL=${SG_URL}
      - SG_SCRIPT=${SG_SCRIPT}
      - SG_KEY=${SG_KEY}
    depends_on:
      - postgres
      - redis
    restart: always

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - sg-cache-api
    restart: always

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    restart: always

  grafana:
    image: grafana/grafana:latest
    volumes:
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    restart: always

volumes:
  postgres-data:
  redis-data:
  prometheus-data:
  grafana-data:
```

### 優點
- ✅ 企業級高可用性
- ✅ 數據持久化和審計追蹤
- ✅ 支持複雜查詢
- ✅ 完整監控和告警
- ✅ 可處理大規模負載
- ✅ 支持歷史數據分析

### 缺點
- ❌ 架構複雜
- ❌ 運維成本高
- ❌ 需要專業團隊維護
- ❌ 基礎設施成本高

---

## 跨平台兼容性保障

### 路徑處理
```python
from pathlib import Path

# 使用 pathlib 處理跨平台路徑
CONFIG_DIR = Path.home() / ".sg_cache"
CONFIG_FILE = CONFIG_DIR / "config.yaml"

# 確保目錄存在
CONFIG_DIR.mkdir(parents=True, exist_ok=True)
```

### 環境變數
```python
import os
from dotenv import load_dotenv

# 使用 .env 文件管理配置
load_dotenv()

SG_URL = os.getenv('SG_URL')
SG_SCRIPT = os.getenv('SG_SCRIPT')
SG_KEY = os.getenv('SG_KEY')
```

### Windows 服務支持
```python
# win_service.py
import win32serviceutil
import win32service
import win32event
import servicemanager

class SGCacheService(win32serviceutil.ServiceFramework):
    _svc_name_ = "ShotGridCache"
    _svc_display_name_ = "ShotGrid Cache Service"
    _svc_description_ = "ShotGrid data caching intermediary"

    def __init__(self, args):
        win32serviceutil.ServiceFramework.__init__(self, args)
        self.stop_event = win32event.CreateEvent(None, 0, 0, None)

    def SvcStop(self):
        self.ReportServiceStatus(win32service.SERVICE_STOP_PENDING)
        win32event.SetEvent(self.stop_event)

    def SvcDoRun(self):
        # 啟動 FastAPI 服務
        import uvicorn
        uvicorn.run("main:app", host="0.0.0.0", port=8000)

if __name__ == '__main__':
    win32serviceutil.HandleCommandLine(SGCacheService)
```

### Linux Systemd 支持
已在各情境的部署配置中提供。

---

## 方案選擇指南

| 考量因素 | 情境一 | 情境二 | 情境三 |
|---------|-------|-------|-------|
| 團隊規模 | < 50 人 | 50-200 人 | 200+ 人 |
| 日訪問量 | < 10K | 10K-100K | > 100K |
| 部署複雜度 | ⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 運維成本 | 低 | 中 | 高 |
| 高可用性 | ❌ | ✅ | ✅✅ |
| 數據持久化 | ❌ | 部分 | ✅ |
| 橫向擴展 | ❌ | ✅ | ✅✅ |
| 審計追蹤 | ❌ | ❌ | ✅ |
| 初期投資 | $0 | $500-2K/月 | $2K-10K/月 |

### 遷移路徑
1. **第一階段**：從情境一開始，驗證概念和需求
2. **第二階段**：當訪問量增長或需要高可用時，升級到情境二
3. **第三階段**：當需要審計、分析或更大規模時，升級到情境三

每個階段的數據模型設計保持兼容性，便於平滑遷移。

---

## 安全性考量（所有情境通用）

### 認證與授權
```python
from fastapi import Security, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt

security = HTTPBearer()

def verify_token(credentials: HTTPAuthorizationCredentials = Security(security)):
    try:
        payload = jwt.decode(
            credentials.credentials,
            SECRET_KEY,
            algorithms=["HS256"]
        )
        return payload
    except jwt.InvalidTokenError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid authentication credentials"
        )

@app.get("/entity/{entity_type}/{entity_id}")
async def get_entity(
    entity_type: str,
    entity_id: int,
    user=Depends(verify_token)
):
    # 檢查用戶權限
    # ...
```

### HTTPS/TLS
- 所有生產環境必須使用 HTTPS
- 使用 Let's Encrypt 或內部 CA 簽發證書
- Nginx 配置示例：
```nginx
server {
    listen 443 ssl http2;
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    # ...
}
```

### 速率限制
```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.get("/query")
@limiter.limit("100/minute")
async def query(request: Request):
    # ...
```

---

## 監控與維護

### 健康檢查端點
```python
@app.get("/health")
async def health_check():
    checks = {
        "api": "ok",
        "redis": await check_redis(),
        "postgres": await check_postgres(),
        "shotgrid": await check_shotgrid()
    }

    if all(v == "ok" for v in checks.values()):
        return {"status": "healthy", "checks": checks}
    else:
        return JSONResponse(
            status_code=503,
            content={"status": "unhealthy", "checks": checks}
        )
```

### Prometheus Metrics
```python
from prometheus_client import Counter, Histogram, generate_latest

REQUEST_COUNT = Counter('sg_cache_requests_total', 'Total requests', ['method', 'endpoint', 'status'])
REQUEST_DURATION = Histogram('sg_cache_request_duration_seconds', 'Request duration')
CACHE_HITS = Counter('sg_cache_hits_total', 'Cache hits', ['cache_type'])
CACHE_MISSES = Counter('sg_cache_misses_total', 'Cache misses', ['cache_type'])

@app.get("/metrics")
async def metrics():
    return Response(generate_latest(), media_type="text/plain")
```

### 日誌策略
```python
import logging
import json

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('/var/log/sg_cache/app.log'),
        logging.StreamHandler()
    ]
)

logger = logging.getLogger(__name__)

# 結構化日誌
logger.info(json.dumps({
    "event": "cache_hit",
    "entity_type": "Shot",
    "entity_id": 12345,
    "response_time_ms": 15.3
}))
```

---

## 結論

三種架構設計提供了從簡單到複雜的漸進式方案：

1. **情境一** 適合快速驗證和小規模使用
2. **情境二** 適合大多數中型工作室的生產環境
3. **情境三** 適合需要企業級功能的大型工作室

所有方案均支持 Linux 和 Windows，使用 Python 生態系統確保跨平台兼容性。建議從情境一開始，根據實際需求逐步升級。

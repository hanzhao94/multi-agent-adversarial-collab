# 电商促销系统 — 极简技术方案

> 设计哲学：简单就是美，能跑就行。YAGNI。

---

## 一、系统架构图

```
┌─────────────┐      ┌──────────────────┐      ┌─────────────┐
│   Admin UI  │─────▶│   Promo Service  │─────▶│   SQLite    │
│  (单页面)    │      │   (唯一后端)      │      │   (单文件)   │
└─────────────┘      └────────┬─────────┘      └─────────────┘
                              │
                              ▼
                     ┌──────────────────┐
                     │   Static Files    │
                     │  (CDN/本地托管)    │
                     └──────────────────┘
```

**一句话架构**：一个后端服务 + 一个数据库文件 + 一个前端页面。没了。

用户请求流程：
1. 商家通过 Admin UI 创建促销规则（满减/折扣/优惠券）
2. Promo Service 校验规则 → 写入 SQLite
3. 前端展示促销列表（纯静态页面 + API 调用）
4. 下单时后端校验订单是否满足促销条件 → 自动计算折扣

---

## 二、模块划分（4 个模块）

| # | 模块 | 职责 | 代码量预估 |
|---|------|------|-----------|
| 1 | **Web Server** | HTTP 路由、请求解析、静态文件服务 | ~300 行 |
| 2 | **Promo Engine** | 促销规则 CRUD + 折扣计算逻辑 | ~500 行 |
| 3 | **Admin UI** | 单页管理界面（创建/编辑/查看促销） | ~400 行 HTML+JS |
| 4 | **Store (DAO)** | SQLite 数据访问封装 | ~200 行 |

**总计预估：~1400 行代码。** 没有多余的。

### 不做的模块（YAGNI 清单）
- ❌ 用户系统（先用硬编码管理员密码，后期再加）
- ❌ 日志系统（std::println 够了）
- ❌ 缓存层（SQLite 够快，没必要）
- ❌ 消息队列（同步就够了）
- ❌ 微服务拆分（4 个模块都塞一个进程里）

---

## 三、技术栈选择

| 层 | 选择 | 理由 |
|----|------|------|
| 后端语言 | **Python** | 生态成熟、代码量少、开发快 |
| Web 框架 | **FastAPI** | 自动 OpenAPI 文档、异步、代码量比 Flask 少 |
| 数据库 | **SQLite** | 零配置、单文件、并发够用（<100 QPS） |
| ORM | **原生 SQL**（sqlite3 标准库） | 表少、SQL 简单，不需要 ORM 的复杂度 |
| 前端 | **HTML + Vanilla JS** | 没有 React/Vue 的构建步骤，一个 HTML 文件搞定 |
| 部署 | **Docker 单容器** | 一条命令启动，10 分钟内搞定 |

**外部依赖清单（3 个，刚好卡上限）：**
1. `fastapi` — Web 框架
2. `uvicorn` — ASGI 服务器
3. `jinja2` — 模板渲染（可选，Admin UI 用纯 HTML 可省掉）

如果 Admin UI 用纯 HTML + fetch API，连 jinja2 都不需要，依赖降到 **2 个**。

---

## 四、数据库设计

仅 2 张表，够用就行。

### 表 1：`promotions`

```sql
CREATE TABLE promotions (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    name        TEXT NOT NULL,           -- 促销名称
    type        TEXT NOT NULL,           -- 'discount' | 'full_reduce' | 'coupon'
    rule_json   TEXT NOT NULL,           -- 规则详情（JSON）
    start_time  TEXT NOT NULL,           -- ISO 8601
    end_time    TEXT NOT NULL,           -- ISO 8601
    status      TEXT DEFAULT 'active',   -- 'active' | 'paused' | 'expired'
    created_at  TEXT DEFAULT (datetime('now'))
);
```

**rule_json 示例：**

```json
// 满减：满 200 减 50
{"type": "full_reduce", "threshold": 200, "reduce": 50}

// 折扣：8 折
{"type": "discount", "rate": 0.8}

// 优惠券：满 100 减 20，限品类
{"type": "coupon", "threshold": 100, "reduce": 20, "category_ids": [1, 3]}
```

> 把复杂规则塞 JSON 里，表结构不变，扩展靠改 JSON schema 而不是改表。

### 表 2：`coupon_codes`（仅优惠券类型需要）

```sql
CREATE TABLE coupon_codes (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    code          TEXT UNIQUE NOT NULL,
    promotion_id  INTEGER REFERENCES promotions(id),
    used          INTEGER DEFAULT 0,     -- 0=未用, 1=已用
    created_at    TEXT DEFAULT (datetime('now'))
);
```

---

## 五、部署方案

### Dockerfile

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 部署步骤（< 10 分钟）

```bash
# 1. 克隆代码（30秒）
git clone <repo>

# 2. 构建镜像（3-5分钟）
docker build -t promo-service .

# 3. 启动（10秒）
docker run -d --name promo \
  -p 8000:8000 \
  -v /data/promo:/app/data \
  promo-service

# 4. 验证（10秒）
curl http://localhost:8000/health
# → {"status": "ok"}
```

SQLite 数据文件通过 volume 挂载到宿主机 `/data/promo/`，容器重建不丢数据。

### 如果不用 Docker（更简单）

```bash
pip install -r requirements.txt
uvicorn main:app --host 0.0.0.0 --port 8000
```

**两条命令，30 秒上线。**

---

## 六、设计理由

### 为什么这么少模块？

**因为促销系统的核心逻辑很简单：** 定义规则 → 存起来 → 算折扣。不需要事件总线、不需要微服务、不需要缓存层。每个额外模块都带来维护成本，而这些成本在当前规模下不会换来任何收益。

### 为什么 SQLite？

- 零运维：不需要安装/配置/备份第三方数据库
- 够用：单机 < 100 QPS 场景下性能绰绰有余
- 备份简单：`cp promo.db promo.db.bak` 就是完整备份
- 迁移无成本：未来真需要 PostgreSQL，改几行连接代码就行

### 为什么不用 Redis？

促销规则变更频率低（一天几次），读取量不大。SQLite 的读性能在这个量级比 Redis 差不了多少，但少了一个外部依赖、少了一套运维。

### 为什么不用 React/Vue？

Admin UI 就是一个表单 + 一个表格。用一个 HTML 文件 + fetch API 搞定，不需要构建工具、不需要打包、不需要 CI/CD 编译前端。

### 什么时候该扩展？

| 触发条件 | 扩展动作 |
|---------|---------|
| QPS > 500 | SQLite → PostgreSQL |
| 需要用户系统 | 加 auth 模块 |
| 前端交互复杂化 | HTML → Vue CDN 版（不用构建） |
| 多实例部署 | Docker Compose + 独立数据库 |

**不到触发条件之前，不加。** 这就是 YAGNI。

---

## 总结

| 指标 | 设计值 |
|------|-------|
| 模块数 | **4 个**（≤5 ✅） |
| 外部依赖 | **2-3 个**（≤3 ✅） |
| 部署时间 | **30秒-5分钟**（≤10min ✅） |
| 代码量 | **~1400 行**（极简 ✅） |
| 核心原则 | **能跑就行，不过度设计** |

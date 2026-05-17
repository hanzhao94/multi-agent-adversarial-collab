# 电商促销系统 — 技术方案设计

> 设计原则：**宁可过度设计，不可设计不足。** 生产级系统必须 Robust，一切围绕可靠性、可扩展性、安全性展开。

---

## 1. 系统架构总览

### 1.1 架构模式

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              接入层 (Edge)                                │
│  CDN(WAF) → API Gateway(Kong/APISIX) → 认证鉴权(OAuth2/JWT) → 限流熔断   │
└────────────────────────────────────────┬─────────────────────────────────┘
                                         │
┌────────────────────────────────────────▼─────────────────────────────────┐
│                        业务服务层 (Microservices)                          │
│                                                                          │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────────────┐    │
│  │ 促销活动   │ │ 优惠券服务 │ │ 秒杀服务   │ │ 积分/奖励服务      │    │
│  │ Campaign   │ │ Coupon     │ │ Flash Sale │ │ Rewards            │    │
│  └──────┬─────┘ └──────┬─────┘ └──────┬─────┘ └────────┬───────────┘    │
│         │              │              │                 │                 │
│  ┌──────▼──────────────▼──────────────▼─────────────────▼───────────┐    │
│  │                  消息队列 (Kafka / RocketMQ)                      │    │
│  └──────────────────────────────┬───────────────────────────────────┘    │
│                                  │                                        │
│  ┌──────────────┐ ┌─────────────▼────────────┐ ┌─────────────────┐       │
│  │ 订单服务     │ │ 库存服务 (扣减/回滚)      │ │ 风控防刷服务    │       │
│  │ Order        │ │ Inventory                 │ │ Anti-Fraud      │       │
│  └──────┬───────┘ └──────────────┬───────────┘ └────────┬────────┘       │
│         │                        │                      │                 │
│  ┌──────▼────────────────────────▼──────────────────────▼───────────┐    │
│  │                  缓存层 (Redis Cluster)                            │    │
│  │  热点数据 / 分布式锁 / 计数器 / 库存预扣减 / 用户参与状态          │    │
│  └──────────────────────────────┬───────────────────────────────────┘    │
└─────────────────────────────────┼───────────────────────────────────────┘
                                  │
┌─────────────────────────────────▼───────────────────────────────────────┐
│                        数据持久层                                        │
│                                                                          │
│  ┌─────────────────┐  ┌──────────────────┐  ┌────────────────────┐      │
│  │ MySQL Cluster   │  │ Elasticsearch    │  │ Object Storage     │      │
│  │ 主从/分库分表    │  │ 活动日志/审计搜索  │  │ 活动素材/图片       │      │
│  └─────────────────┘  └──────────────────┘  └────────────────────┘      │
│  ┌─────────────────┐                                                     │
│  │ ClickHouse      │  — 实时分析 & 数据看板                               │
│  └─────────────────┘                                                     │
└──────────────────────────────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────────────────────────────┐
│                        基础设施 & 运维                                    │
│                                                                          │
│  K8s(HPA/VPA) │ Prometheus + Grafana │ ELK │ SkyWalking │ AlertManager │
│  CI/CD(GitHub Actions) │ 蓝绿/金丝雀部署 │ 混沌工程(Chaos Mesh)          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 1.2 架构要点

| 层级 | 职责 | 核心理念 |
|------|------|----------|
| Edge/网关 | 统一入口、WAF、限流、认证 | 第一道防线 |
| 业务微服务 | 促销、券、秒杀、库存、订单、风控 | 单一职责，独立部署 |
| 消息中间件 | 异步解耦、削峰填谷、事件溯源 | 可靠传递，至少一次 |
| 缓存 | 热点读取、分布式锁、库存预扣减 | 缓存是盾不是源 |
| 存储 | MySQL(核心) + ES(搜索) + ClickHouse(分析) | 读写分离，冷热分离 |

---

## 2. 模块划分

### 2.1 服务清单

| # | 服务名 | 职责 | 端口 |
|---|--------|------|------|
| 1 | `api-gateway` | 路由转发、鉴权、限流、灰度发布 | 8000 |
| 2 | `campaign-service` | 促销活动CRUD、活动规则引擎、排期管理 | 8001 |
| 3 | `coupon-service` | 优惠券模板、发券、核销、过期处理 | 8002 |
| 4 | `flash-sale-service` | 秒杀排队、库存预扣减、防并发超卖 | 8003 |
| 5 | `inventory-service` | 库存扣减、回滚、安全库存预警、对账 | 8004 |
| 6 | `order-service` | 促销订单创建、金额计算、超时取消 | 8005 |
| 7 | `pricing-engine` | 优惠叠加规则、价格计算引擎、最低价校验 | 8006 |
| 8 | `fraud-detection` | 风控规则、黑产识别、行为分析、封禁 | 8007 |
| 9 | `notification-service` | 短信/推送/邮件通知、模板管理 | 8008 |
| 10 | `analytics-service` | 实时数据看板、转化漏斗、AB测试 | 8009 |

### 2.2 服务间通信

```
同步: REST (HTTP/2) — 查询类、强一致性要求场景
异步: Kafka — 订单创建、库存扣减、通知触发、审计日志
  ├─ topic: order.created
  ├─ topic: inventory.deducted
  ├─ topic: coupon.issued
  ├─ topic: coupon.consumed
  ├─ topic: fraud.alert
  └─ topic: audit.log
```

---

## 3. 技术栈选择

### 3.1 核心选型

| 类别 | 选型 | 理由 |
|------|------|------|
| 语言/框架 | Java 21 + Spring Boot 3.2 + Spring Cloud 2023 | 生态成熟，团队友好，Spring Cloud Alibaba 全家桶 |
| 网关 | APISIX (替代 Kong) | 性能更高，动态路由，热更新，Lua 插件扩展 |
| 消息队列 | Apache RocketMQ 5.x | 事务消息、顺序消息、延迟消息，电商场景最优 |
| 缓存 | Redis 7 Cluster + Redisson | 分布式锁、Stream、Bloom Filter |
| 数据库 | MySQL 8.0 (InnoDB Cluster / Group Replication) | 自动故障切换，强一致 |
| 搜索引擎 | Elasticsearch 8.x | 活动检索、审计日志、操作回溯 |
| OLAP | ClickHouse | 实时数据分析、大促大屏 |
| 配置中心 | Nacos 2.x | 服务发现 + 配置管理 + 灰度发布 |
| 链路追踪 | Apache SkyWalking | 无侵入，Java 生态最好 |
| 监控 | Prometheus + Grafana + AlertManager | 事实标准，生态丰富 |
| 容器编排 | Kubernetes 1.28+ | HPA/VPA、亲和性、多可用区 |
| CI/CD | ArgoCD + GitHub Actions | GitOps，不可变基础设施 |
| 混沌工程 | Chaos Mesh | 定期故障注入，验证容灾 |

### 3.2 中间件部署规格（生产级）

| 组件 | 部署模式 | 最低规格 |
|------|----------|----------|
| APISIX | 3节点 (至少) | 4C8G × 3 |
| RocketMQ | 2主2从 + 2 NameServer | 8C16G × 4 |
| Redis Cluster | 6节点 (3主3从) | 4C8G × 6 |
| MySQL | 1主2从 + MGR | 8C32G × 3 |
| Nacos | 3节点集群 | 4C8G × 3 |
| Kafka (备用) | 3 Broker + 3 ZK | 4C16G × 3 |

---

## 4. 数据库设计

### 4.1 核心表结构

```sql
-- ==========================================
-- 促销活动表 (campaign-service)
-- ==========================================
CREATE TABLE campaign (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    campaign_no     VARCHAR(32)  NOT NULL UNIQUE,    -- 活动编号: CP20260518000001
    campaign_type   TINYINT      NOT NULL,            -- 1=满减 2=打折 3=秒杀 4=拼团 5=预售
    name            VARCHAR(128) NOT NULL,
    description     TEXT,
    status          TINYINT      NOT NULL DEFAULT 0,  -- 0=草稿 1=待开始 2=进行中 3=已结束 4=已取消
    start_time      DATETIME     NOT NULL,
    end_time        DATETIME     NOT NULL,
    rule_json       JSON         NOT NULL,            -- { "threshold": 100, "discount": 20, "max_discount": 50 }
    stacking_rule   TINYINT      NOT NULL DEFAULT 0,  -- 0=不可叠加 1=可叠加优惠券 2=可叠加全部
    budget_amount   DECIMAL(12,2) DEFAULT NULL,       -- 活动预算上限(防超发)
    used_amount     DECIMAL(12,2) DEFAULT 0,
    participant_limit BIGINT     DEFAULT NULL,        -- 参与人数上限
    participant_count BIGINT     DEFAULT 0,
    created_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    created_by      VARCHAR(64)  NOT NULL,
    version         INT          NOT NULL DEFAULT 0,  -- 乐观锁
    INDEX idx_status_time (status, start_time, end_time),
    INDEX idx_type_status (campaign_type, status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='促销活动表';

-- ==========================================
-- 优惠券模板表 (coupon-service)
-- ==========================================
CREATE TABLE coupon_template (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    template_no     VARCHAR(32)  NOT NULL UNIQUE,
    campaign_id     BIGINT       NOT NULL,
    coupon_type     TINYINT      NOT NULL,            -- 1=满减 2=折扣 3=运费券 4=兑换券
    name            VARCHAR(64)  NOT NULL,
    face_value      DECIMAL(10,2) NOT NULL,           -- 面值
    discount_rate   DECIMAL(4,2) DEFAULT NULL,        -- 折扣率(仅type=2)
    min_amount      DECIMAL(10,2) DEFAULT NULL,       -- 使用门槛
    max_discount    DECIMAL(10,2) DEFAULT NULL,       -- 最大抵扣
    total_count     INT          NOT NULL,
    issued_count    INT          DEFAULT 0,
    used_count      INT          DEFAULT 0,
    valid_days      INT          DEFAULT NULL,        -- 领取后有效天数
    fixed_start     DATETIME     DEFAULT NULL,        -- 固定生效时间
    fixed_end       DATETIME     DEFAULT NULL,        -- 固定过期时间
    per_user_limit  INT          DEFAULT 1,           -- 每人限领
    scope_type      TINYINT      DEFAULT 0,           -- 0=全场 1=品类 2=商品
    scope_ids       JSON         DEFAULT NULL,        -- 适用范围ID列表
    status          TINYINT      DEFAULT 1,           -- 0=停用 1=启用
    created_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_campaign (campaign_id),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ==========================================
-- 用户优惠券表
-- ==========================================
CREATE TABLE user_coupon (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id         BIGINT       NOT NULL,
    template_id     BIGINT       NOT NULL,
    coupon_code     VARCHAR(32)  NOT NULL UNIQUE,     -- 唯一券码
    status          TINYINT      NOT NULL DEFAULT 0,  -- 0=未使用 1=已使用 2=已过期 3=已锁定
    order_id        BIGINT       DEFAULT NULL,        -- 使用订单号
    valid_from      DATETIME     NOT NULL,
    valid_to        DATETIME     NOT NULL,
    used_at         DATETIME     DEFAULT NULL,
    created_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user_status (user_id, status),
    INDEX idx_valid_to (valid_to, status),            -- 过期定时任务扫描
    INDEX idx_code (coupon_code)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ==========================================
-- 秒杀库存表 (flash-sale-service)
-- ==========================================
CREATE TABLE flash_sale_stock (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    campaign_id     BIGINT       NOT NULL,
    sku_id          BIGINT       NOT NULL,
    total_stock     INT          NOT NULL,
    available_stock INT          NOT NULL,            -- 可用库存(= total - locked - sold)
    locked_stock    INT          DEFAULT 0,           -- 锁定中(下单未支付)
    sold_stock      INT          DEFAULT 0,
    version         INT          NOT NULL DEFAULT 0,
    created_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_campaign_sku (campaign_id, sku_id),
    INDEX idx_available (campaign_id, available_stock)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ==========================================
-- 幂等表 (所有写操作共用)
-- ==========================================
CREATE TABLE idempotent_record (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    biz_type        VARCHAR(32)  NOT NULL,            -- coupon_issue / order_create / stock_deduct
    biz_key         VARCHAR(64)  NOT NULL,            -- 业务唯一键
    biz_result      TEXT,                             -- 执行结果(缓存)
    status          TINYINT      DEFAULT 1,           -- 1=成功 0=失败
    created_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    expire_at       DATETIME     NOT NULL,
    UNIQUE KEY uk_biz (biz_type, biz_key),
    INDEX idx_expire (expire_at)                      -- 定时清理
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ==========================================
-- 活动审计日志 (写入ES，MySQL保留近期)
-- ==========================================
CREATE TABLE campaign_audit_log (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    campaign_id     BIGINT       NOT NULL,
    operator        VARCHAR(64)  NOT NULL,
    action          VARCHAR(32)  NOT NULL,
    before_state    JSON,
    after_state     JSON,
    ip_address      VARCHAR(45),
    user_agent      VARCHAR(256),
    created_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_campaign_time (campaign_id, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 4.2 分库分表策略

| 表 | 分片键 | 分片数 | 说明 |
|---|--------|--------|------|
| `user_coupon` | `user_id % 64` | 64表 | 按用户维度查询最多 |
| `flash_sale_stock` | `campaign_id % 32` | 32表 | 活动维度隔离 |
| `campaign_audit_log` | 按月分区 | 分区表 | 时间范围查询，定期归档 |
| `order_*` (外部系统) | `user_id % 128` | 128表 | 订单量大，需要预留 |

### 4.3 缓存策略

```
Redis Key 设计:
─────────────────────────────────────────────────────────────────────
活动详情:     campaign:detail:{campaign_no}          TTL: 活动结束
活动规则:     campaign:rule:{campaign_no}            TTL: 24h (可配)
优惠券模板:   coupon:template:{template_no}          TTL: 24h
用户券列表:   user:coupons:{user_id}                 TTL: 1h
秒杀库存:     flash:stock:{campaign_id}:{sku_id}     无TTL, DB为准
秒杀排队:     flash:queue:{campaign_id}:{sku_id}     Set, ZSet按时间戳
用户参与:     flash:participated:{campaign_id}:{user_id}  TTL: 活动结束
防刷计数器:   fraud:count:{action}:{user_id}:{date}  TTL: 1d
分布式锁:     lock:{biz_type}:{resource_id}          TTL: 5s + watchdog
幂等结果:     idempotent:{biz_type}:{biz_key}        TTL: 24h
```

---

## 5. 高可用方案 (99.99% SLA)

### 5.1 SLA 计算

```
99.99% = 全年停机时间 ≤ 52.56 分钟
分解到各组件可用性要求:
  - API Gateway: 99.995%  (多可用区部署)
  - 业务服务:   99.99%   (至少2副本 + 自动故障转移)
  - 数据库:     99.99%   (MGR自动切换 + 延迟<10s)
  - Redis:      99.99%   (Cluster + Sentinel, 3主3从)
  - 消息队列:   99.99%   (双主双从, 同步复制)

综合可用性 = ∏(各组件可用性) ≥ 99.99%
```

### 5.2 多可用区部署

```
可用区 A                    可用区 B                    可用区 C
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│ APISIX × 2      │       │ APISIX × 1      │       │ (冷备)          │
│ App Pod × 3     │       │ App Pod × 3     │       │                 │
│ MySQL Primary   │◄─────►│ MySQL Slave     │       │ MySQL Slave     │
│ Redis Master×2  │◄─────►│ Redis Slave×2   │       │ Redis Slave×1   │
│ RocketMQ M×2    │◄─────►│ RocketMQ S×2    │       │                 │
│ Nacos × 2       │◄─────►│ Nacos × 1       │       │                 │
└────────┬────────┘       └────────┬────────┘       └────────┬────────┘
         │        K8s Cluster (跨AZ)         │               │
         └──────────────────┬────────────────┘               │
                            ▼                                │
                    Global DNS (多AZ解析)                     │
                            └────────────────────────────────┘
```

### 5.3 故障转移策略

| 组件 | 故障场景 | 恢复机制 | RTO | RPO |
|------|----------|----------|-----|-----|
| APISIX | 单节点宕机 | 健康检查摘流, 剩余节点接管 | < 10s | 0 |
| 业务服务 | Pod崩溃 | K8s自动重启 + HPA扩容 | < 30s | 0 |
| MySQL | 主节点宕机 | MGR自动选主 + VIP漂移 | < 10s | < 1s |
| Redis | Master宕机 | Sentinel自动切换 | < 3s | 0 |
| RocketMQ | Broker宕机 | 从节点升主, 生产者重试 | < 30s | 0 (同步) |
| 整个AZ故障 | 云可用区宕机 | DNS切换流量到可用AZ | < 60s | < 5s |

### 5.4 降级策略 (核心!)

```yaml
降级分级 (Degradation Levels):
  L0 - 正常: 全功能可用
  L1 - 轻度降级:
    - 关闭非核心功能: 活动推荐、个性化排序
    - 降级ES搜索为本地缓存
    - 关闭实时分析看板
  L2 - 中度降级:
    - 关闭优惠券叠加规则计算(使用预设缓存结果)
    - 秒杀改为纯Redis库存校验(跳过DB校验)
    - 通知异步化(从同步改消息队列)
  L3 - 重度降级:
    - 秒杀入口关闭(保护库存)
    - 仅允许已登录用户访问
    - 关闭所有写操作(只读模式)
    - 返回静态缓存页面
  L4 - 熔断:
    - 全量熔断, 返回服务维护页
    - 仅保留心跳检查端点

触发条件:
  - CPU > 85% 持续30s → L1
  - 错误率 > 5% 持续10s → L2
  - 数据库连接池 > 90% 持续5s → L3
  - 核心服务全部不可用 → L4

恢复策略:
  - 逐层恢复, 每层间隔 ≥ 60s
  - 人工确认后恢复L3以上
```

### 5.5 熔断器配置

```java
// Resilience4j 配置 (每个服务独立)
resilience4j.circuitbreaker:
  instances:
    inventoryService:
      failureRateThreshold: 50          # 失败率超过50%熔断
      slowCallRateThreshold: 80         # 慢请求超过80%熔断
      slowCallDurationThreshold: 2000   # > 2s 视为慢请求
      slidingWindowSize: 20             # 滑动窗口大小
      minimumNumberOfCalls: 10          # 最小请求数
      permittedNumberOfCallsInHalfOpenState: 5
      waitDurationInOpenState: 30s      # 熔断后等待30s再尝试半开
      recordExceptions:
        - java.sql.SQLException
        - java.net.ConnectException
        - org.springframework.web.client.ResourceAccessException

    couponService:
      failureRateThreshold: 40
      waitDurationInOpenState: 20s
      slidingWindowSize: 50

    flashSaleService:
      failureRateThreshold: 30          # 秒杀更敏感
      waitDurationInOpenState: 60s      # 秒杀熔断后更长恢复时间
```

---

## 6. 监控告警方案

### 6.1 监控四层体系

```
┌─────────────────────────────────────────────────────────────┐
│ L1: 基础设施监控 (Infrastructure)                             │
│   CPU / Memory / Disk / Network / Pod状态 / 节点状态           │
├─────────────────────────────────────────────────────────────┤
│ L2: 中间件监控 (Middleware)                                   │
│   MySQL QPS/连接数/主从延迟/慢SQL                              │
│   Redis 内存/命中率/连接数/Eviction                           │
│   RocketMQ 堆积量/TPS/Consumer延迟                            │
├─────────────────────────────────────────────────────────────┤
│ L3: 应用监控 (APM)                                           │
│   QPS / RT(p50/p95/p99) / 错误率 / JVM / GC / 线程池         │
│   SkyWalking 全链路追踪                                        │
├─────────────────────────────────────────────────────────────┤
│ L4: 业务监控 (Business)                                      │
│   发券量 / 核销率 / 秒杀成功率 / 超卖次数 / 转化率 / GMV        │
│   异常下单率 / 黑产拦截率 / 活动参与度                         │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 告警规则 (AlertManager)

| 级别 | 规则 | 通知方式 | 响应时间 |
|------|------|----------|----------|
| P0-致命 | 核心服务全部不可用 / 数据库宕机 / 超卖 | 电话 + 短信 + 企业微信 | 5min |
| P1-严重 | 错误率>5% / 数据库主从延迟>5s / MQ堆积>10万 | 企业微信 + 邮件 | 15min |
| P2-警告 | P99延迟>2s / CPU>85% / 内存>90% / 连接池>80% | 企业微信 | 30min |
| P3-提示 | 慢SQL>1s / 缓存命中率<90% / GC频繁 | Dashboard | 4h |

### 6.3 关键监控大盘

```
Dashboard 1: 促销活动总览
  ├─ 各活动参与UV/PV实时曲线
  ├─ 优惠券领取量/核销率/剩余量
  ├─ 秒杀商品库存消耗速度
  └─ 实时GMV & 转化率

Dashboard 2: 系统健康度
  ├─ 各服务QPS/RT/错误率
  ├─ 数据库连接池/慢SQL/主从延迟
  ├─ Redis内存/命中率/大Key
  └─ MQ堆积量/消费延迟

Dashboard 3: 安全与风控
  ├─ 黑产请求拦截数
  ├─ 异常IP Top20
  ├─ 刷单检测命中率
  └─ 验证码触发率
```

### 6.4 日志体系

```
日志采集: Filebeat → Kafka → Logstash → Elasticsearch
日志规范:
  - 格式: JSON (方便解析)
  - 字段: traceId, spanId, service, level, timestamp, message, bizType, userId
  - 级别: ERROR(告警) > WARN(监控) > INFO(审计) > DEBUG(开发)
  - 敏感数据: 手机号/身份证/卡号 必须脱敏
  - 保留策略: ES热数据7天 → 冷存储90天 → 归档删除
```

---

## 7. 安全方案

### 7.1 防刷体系

```
四层防护网:
┌─────────────────────────────────────────────────────┐
│ Layer 1: 网关层                                      │
│   - WAF规则 (SQL注入/XSS/CC攻击)                     │
│   - IP频率限制: 全局 1000req/min/IP                  │
│   - UA校验: 拦截已知爬虫UA                           │
│   - 设备指纹: Canvas/WebGL指纹                       │
├─────────────────────────────────────────────────────┤
│ Layer 2: 应用层                                      │
│   - 用户级限流: 每人每分钟领券≤5次, 下单≤3次          │
│   - Redis计数器 + 滑动窗口限流                        │
│   - 验证码: 异常行为触发滑块/点选验证                  │
│   - 行为分析: 鼠标轨迹/点击间隔/页面停留              │
├─────────────────────────────────────────────────────┤
│ Layer 3: 业务层                                      │
│   - 每人限购/限领: DB + Redis双重校验                 │
│   - 黑名单: 设备ID / IP / 手机号                      │
│   - 同设备多账号检测                                   │
│   - 异常IP关联分析                                    │
├─────────────────────────────────────────────────────┤
│ Layer 4: 数据层                                      │
│   - 实时风控规则引擎 (Drools)                        │
│   - 机器学习异常检测 (孤立森林/One-Class SVM)          │
│   - 图神经网络: 识别团伙刷单                          │
│   - 事后审计: ClickHouse离线分析                     │
└─────────────────────────────────────────────────────┘
```

### 7.2 限流策略

```yaml
限流配置 (Sentinel / APISIX):
  全局限流:
    - 总QPS: 100,000 (可动态调整)
    - 单IP QPS: 100
    - 单用户 QPS: 50

  秒杀接口限流:
    - 令牌桶: 每秒5000令牌
    - 排队机制: 超出进入虚拟队列
    - 拒绝策略: 返回"活动太火爆,请稍后再试"
    - 排队超时: 10s自动拒绝

  领券接口限流:
    - 滑动窗口: 每用户每分钟5次
    - 全局QPS: 5000
    - 超限时: 返回友好提示
```

### 7.3 幂等性设计

```java
// 所有写操作必须幂等! 三重保障:

// 1. 业务层: 幂等Token
String idempotentKey = "coupon_issue:" + userId + ":" + templateId + ":" + requestId;
Boolean isFirst = redis.opsForValue()
    .setIfAbsent(idempotentKey, "1", 24, TimeUnit.HOURS);
if (!isFirst) {
    return cache.get(idempotentKey + ":result"); // 返回缓存结果
}

// 2. 数据库层: 唯一索引
// CREATE UNIQUE INDEX uk_biz ON idempotent_record(biz_type, biz_key);

// 3. 消息层: 消息去重
// RocketMQ: 消息Key = bizKey, 消费端用Key做幂等判断
// Kafka: 开启幂等生产者 (enable.idempotence=true)

// 4. 分布式锁: 关键操作加锁 (Redisson)
RLock lock = redisson.getLock("lock:stock_deduct:" + skuId);
try {
    if (lock.tryLock(3, 10, TimeUnit.SECONDS)) {
        // 扣减库存
    }
} finally {
    if (lock.isHeldByCurrentThread()) lock.unlock();
}
```

### 7.4 数据安全

```
- 传输加密: 全站HTTPS (TLS 1.3)
- 存储加密: 用户手机号/身份证 AES-256加密存储
- 密钥管理: KMS / HashiCorp Vault
- 数据库审计: 所有DML操作记录审计日志
- 数据脱敏: 日志/监控/开发环境 全量脱敏
- 权限控制: RBAC + 数据行级权限
- 防SQL注入: MyBatis参数绑定 + 禁止拼接
- 防越权: 每次操作校验userId归属
```

### 7.5 秒杀防超卖 (核心方案)

```java
/**
 * 秒杀库存扣减流程 — 三层防护
 */
public boolean deductStock(Long campaignId, Long skuId, int quantity) {
    // 第一层: Redis原子扣减 (高性能)
    String key = "flash:stock:" + campaignId + ":" + skuId;
    Long remaining = redis.opsForValue().decrement(key, quantity);
    if (remaining == null || remaining < 0) {
        redis.opsForValue().increment(key, quantity); // 回滚
        return false; // 库存不足
    }

    // 第二层: 异步写入DB (可靠)
    kafkaTemplate.send("inventory.deduct", new StockDeductEvent(
        campaignId, skuId, quantity, System.currentTimeMillis()
    ));

    // 第三层: 定时对账 (兜底)
    // 每5秒比对 Redis库存 与 DB库存, 发现不一致自动修复
    return true;
}

// 对账任务
@Scheduled(fixedRate = 5000)
public void reconcileStock() {
    // SELECT sku_id, available_stock FROM flash_sale_stock WHERE ...
    // 对比 Redis, 差异超过阈值则告警并修复
}
```

---

## 8. 部署方案

### 8.1 K8s 部署架构

```yaml
# deployment.yaml — 以 campaign-service 为例
apiVersion: apps/v1
kind: Deployment
metadata:
  name: campaign-service
  namespace: promotion-prod
spec:
  replicas: 3                          # 最小3副本
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1                # 滚动更新最多停1个
      maxSurge: 1                      # 最多超配1个
  template:
    spec:
      topologySpreadConstraints:       # 跨AZ打散
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
      containers:
        - name: campaign-service
          image: registry/campaign-service:${VERSION}
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8001
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8001
            initialDelaySeconds: 10
            periodSeconds: 5
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"
            - name: JAVA_OPTS
              value: "-XX:+UseZGC -Xmx3g -Xms3g"
---
# HPA — 自动扩缩容
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: campaign-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: campaign-service
  minReplicas: 3
  maxReplicas: 20                    # 大促时可自动扩到20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
---
# PDB — 保证最小可用
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: campaign-service-pdb
spec:
  minAvailable: 2                    # 维护时至少保留2个Pod
  selector:
    matchLabels:
      app: campaign-service
---
# 服务网格 (Istio) — 流量管理
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: campaign-service-vs
spec:
  hosts:
    - campaign-service
  http:
    - route:
        - destination:
            host: campaign-service
            subset: v1
          weight: 95
        - destination:
            host: campaign-service
            subset: v2
          weight: 5                 # 5%灰度流量
---
# 网络策略 — 最小权限
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: campaign-service-netpol
spec:
  podSelector:
    matchLabels:
      app: campaign-service
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api-gateway
      ports:
        - port: 8001
          protocol: TCP
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: mysql
      ports:
        - port: 3306
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - port: 6379
    - to:
        - podSelector:
            matchLabels:
              app: rocketmq
      ports:
        - port: 9876
```

### 8.2 部署流程 (CI/CD)

```
Developer → Git Push
    │
    ▼
GitHub Actions
    ├─ 代码检查 (SonarQube)
    ├─ 单元测试 (覆盖率 ≥ 80%)
    ├─ 集成测试 (H2 + Testcontainers)
    ├─ 构建镜像 (多阶段构建, 安全扫描)
    └─ 推送镜像仓库 (Harbor)
    │
    ▼
ArgoCD 检测变更
    ├─ 同步到 Staging 环境
    ├─ 自动冒烟测试
    ├─ 通过 → 金丝雀 5% → 20% → 50% → 100%
    └─ 任一步失败 → 自动回滚
    │
    ▼
生产环境
    ├─ 蓝绿部署 (核心服务)
    ├─ 滚动更新 (普通服务)
    └─ 手动确认后清理旧版本
```

### 8.3 环境规划

| 环境 | 用途 | 副本数 | 数据 |
|------|------|--------|------|
| Dev | 开发调试 | 1 | Mock/本地DB |
| Staging | 集成测试/性能测试 | 2 | 脱敏生产数据 |
| Pre-Prod | 上线前验证(与Prod同配置) | 3 | 脱敏生产数据 |
| Prod | 生产环境 | ≥3 | 生产数据 |

### 8.4 灾备方案

```
RPO (数据丢失容忍): ≤ 1秒
RTO (恢复时间): ≤ 60秒

备份策略:
  - MySQL: 每日全量备份 + Binlog实时备份 (保留30天)
  - Redis: RDB每小时 + AOF每秒 (保留7天)
  - ES: 每日快照到对象存储 (保留30天)
  - 配置: Git管理, 随时可回滚

容灾演练 (每月一次):
  - 主库宕机切换
  - AZ故障切换
  - Redis集群重建
  - MQ堆积恢复
```

---

## 9. 设计理由

### 9.1 为什么这样选?

| 决策 | 选择 | 为什么不选其他 |
|------|------|----------------|
| Java + Spring Boot | 成熟稳定, 团队熟悉度高, 生态最全 | Go性能更好但生态不如Java, Node.js适合IO密集不适合计算密集 |
| RocketMQ > Kafka | 电商场景: 事务消息保证发券+订单一致性, 延迟消息支持订单超时取消 | Kafka吞吐量更高但延迟消息/事务消息支持弱 |
| Redis Cluster | 高可用+水平扩展, 分布式锁支持好 | Memcached无持久化, Redis Sentinel不能水平扩展 |
| MySQL Group Replication | 自动故障切换, 强一致模式(SINGLE-PRIMARY) | 主从复制有数据丢失风险, Galera配置复杂 |
| APISIX > Kong | 性能更好(基准测试高30%), 动态配置无需重启, 插件开发更简单 | Kong生态更大但性能略逊, 热更新能力差 |
| ClickHouse | 实时分析性能碾压, 大促看板秒级响应 | Doris也不错但生态不如CH, ES分析性能不够 |
| K8s HPA + 金丝雀 | 自动弹性伸缩, 灰度发布零停机 | 虚拟机部署扩缩容慢, 蓝绿切换有瞬断风险 |

### 9.2 "宁可过度设计" 体现在哪

1. **双写校验**: 秒杀库存同时写Redis和DB, 定时对账兜底
2. **三重幂等**: 业务层(Redis) + 数据层(唯一索引) + 消息层(Key去重)
3. **四层降级**: L0→L4逐级保护, 不是简单的"开/关"
4. **四层防刷**: 网关→应用→业务→数据, 纵深防御
5. **多可用区**: 不是单机部署, 跨3个可用区
6. **PDB约束**: K8s维护时保证最小副本数
7. **混沌工程**: 主动故障注入, 不是等真出事了才知道
8. **金丝雀+自动回滚**: 不是直接全量发布
9. **网络策略**: K8s NetworkPolicy限制Pod间通信, 最小权限
10. **审计日志**: 每个操作都有before/after, 可回溯

### 9.3 成本考量

```
过度设计的代价 = 更高的服务器成本
但促销系统的代价 = 超卖/资损/品牌信任崩塌

一次超卖事故的损失:
  - 直接资损: 可能数十万到数百万
  - 品牌损失: 用户信任不可估量
  - 公关成本: 负面舆情处理
  - 技术债务: 事后补救的混乱

所以: 服务器成本 << 事故成本
      过度设计 > 设计不足

但"过度"≠"无脑堆机器":
  - 用合理的架构降低单机压力
  - 用缓存减少对DB的冲击
  - 用异步解耦降低同步阻塞
  - 用限流熔断保护系统不被击穿
  - 这些都是"聪明的过度设计"
```

---

## 附录: 大促压测清单

```
□ 单机压测: 每个服务独立压测, 找出性能瓶颈
□ 链路压测: 全链路压测, 模拟真实用户行为
□ 峰值压测: 目标QPS的3倍压力, 验证熔断触发
□ 降级验证: 手动触发降级, 验证各层级效果
□ 故障注入: 随机Kill Pod/断网/慢SQL, 验证恢复
□ 数据校验: 压测后校验库存/优惠券/订单数据一致性
□ 监控验证: 确认所有告警规则正常触发
□ 回滚演练: 模拟需要回滚场景, 验证回滚流程
```

> **设计者注**: 本方案以99.99% SLA为底线, 所有设计决策围绕"生产不出事故"展开。每一个冗余、每一层防护、每一个降级策略, 都是在为"万一大促峰值超出预期"做准备。宁可备而不用, 不可用而无备。

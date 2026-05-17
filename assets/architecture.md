# Joint Solution Architecture

```
┌──────────────┐      ┌───────────────────────────────┐      ┌──────────────┐
│   Admin UI   │─────▶│       Promo Service            │─────▶│  PostgreSQL  │
│  Modular JS  │      │   (Single process, 3 modules)  │      │  (Instance)   │
└──────────────┘      │                               │      └──────────────┘
                      │  ┌─────────────────────────┐  │
                      │  │  Redis (optional)        │  │
                      │  │  - Distributed locks     │  │
                      │  │  - Counters / rate limit │  │
                      │  └─────────────────────────┘  │
                      └───────────────────────────────┘
```

## Module Breakdown

| # | Module | Responsibility | Lines |
|---|--------|---------------|-------|
| 1 | API Layer | HTTP routing, JWT auth, rate limiting, validation | ~400 |
| 2 | Promo Engine | Rule CRUD, discount calc, idempotency, concurrency safety | ~600 |
| 3 | Admin UI | HTML + ES6 modules (no build step) | ~500 |

## Evolution Path

```
Phase 1 (MVP — 1-2 weeks):
  3-module monolith + PostgreSQL + 7 core capabilities
  Deploy: Docker Compose single machine

Phase 2 (Hardening — 2-4 weeks):
  + Redis + basic monitoring (Prometheus + Grafana)
  + Rate limiting + 3-level degradation
  2 machines (active-standby)

Phase 3 (Scale — on demand):
  QPS > 1000 → Split services, K8s
  QPS > 5000 → Read-write splitting
  Attack → Rule engine → ML
  Analytics → ES / ClickHouse
```

# Multi-Agent Adversarial Collaboration Experiment

> When two AI agents with **conflicting goals** try to design the same system, what happens?

## Overview

This experiment tests whether multiple AI agents can autonomously coordinate when given the same task but **conflicting optimization objectives**.

**Task:** Design a technical architecture for an e-commerce promotion system.

| Agent | Goal | Constraints |
|-------|------|-------------|
| 🔵 Minimalist | Minimize complexity | ≤5 modules, ≤3 deps, ≤10min deploy |
| 🔴 Perfectionist | Maximize reliability | 99.99% SLA, HA, monitoring, security |

## Key Finding

**Both agents converged to the same conclusion independently after adversarial review.**

They discovered that the real enemy wasn't each other's design — it was **undefined scene constraints**. Once context was added (< 10k orders/day, 1-3 person team, low budget), the gap narrowed dramatically.

## Results

### Before (Divergence)

| Dimension | 🔵 Minimalist | 🔴 Perfectionist | Gap |
|-----------|--------------|------------------|-----|
| Modules | 4 | 10 | 🔴🔴🔴 |
| Dependencies | 2-3 | 13+ | 🔴🔴🔴 |
| Code lines | ~1,400 | ~50,000 | 🔴🔴🔴 |
| Deploy time | 30s | Hours | 🔴🔴 |
| Database | SQLite | MySQL cluster + ES + ClickHouse | 🔴🔴🔴 |
| SLA | None | 99.99% | 🔴🔴🔴 |

### After (Convergence)

| Dimension | Joint Solution | Notes |
|-----------|---------------|-------|
| Modules | **3** | API Layer + Promo Engine + Admin UI |
| Dependencies | **4** | FastAPI + uvicorn + psycopg2 + redis |
| Code lines | **~1,500** | Includes all 7 core security capabilities |
| Deploy time | **2 min** | Docker Compose |
| Database | **PostgreSQL** (single instance) | |
| SLA | **99.9%** | Reasonable target for promotion system |

### The "Precision Design" Principle

> **Minimalism is not cutting all safeguards.**
> **Perfection is not stacking all components.**
> **Precision is matching architectural complexity to actual scene constraints.**

## File Structure

```
multi-agent-adversarial-collab/
├── README.md                    # This file
├── experiment_design.md         # Experiment design & methodology
├── joint_solution.md            # Final joint solution (arbitrated)
├── agents/
│   ├── minimalist/
│   │   ├── design.md            # Minimalist's original proposal
│   │   └── review.md            # Minimalist's review of Perfectionist
│   └── perfectionist/
│       ├── design.md            # Perfectionist's original proposal
│       └── review.md            # Perfectionist's review of Minimalist
└── assets/
    └── architecture.png         # Joint solution architecture diagram
```

## Experiment Details

See [experiment_design.md](experiment_design.md) for full methodology, evaluation metrics, and conclusions.

See [joint_solution.md](joint_solution.md) for the final joint technical proposal.

## Core Insights

1. **Adversarial goals ≠ impossible collaboration** — Both agents found reasonable compromises
2. **The biggest enemy is undefined context** — Without scene constraints, both designs were fatally flawed
3. **Core security costs less than you think** — 7 essential capabilities = ~200 extra lines of code
4. **Multi-agent value = mutual restraint + blind-spot discovery** — Not "two heads are better than one"

## Run Your Own

This experiment framework can be reused:
1. Define a task
2. Create 2+ agents with conflicting goals
3. Round 1: Independent proposals
4. Round 2: Cross-review
5. Arbitrate → joint solution

## License

MIT

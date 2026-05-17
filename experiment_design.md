# Experiment Design: Adversarial Multi-Agent Collaboration

## Hypothesis

Two agents assigned the same task with **conflicting optimization goals** will:
1. Discover the conflict
2. Negotiate / compromise / compete
3. Produce a joint solution

## Task: E-commerce Promotion System

### Agent A ("Minimalist")
- **Goal:** Minimize system complexity
- **Constraints:** ≤5 modules, ≤3 dependencies, ≤10min deployment
- **Values:** Simple is beautiful, YAGNI principle

### Agent B ("Perfectionist")
- **Goal:** Maximize reliability and scalability
- **Constraints:** 99.99% SLA, HA, monitoring, degradation, scaling
- **Values:** Production-grade must be robust, better over-designed than under-designed

## Process

| Step | Action |
|------|--------|
| Round 1 | Independent proposals |
| Round 2 | Cross-review each other's proposal |
| Round 3 | Arbitration → joint solution |

## Evaluation Metrics

| Metric | Description |
|--------|-------------|
| Conflict Discovery Time | How quickly they realize goals conflict |
| Negotiation Rounds | Rounds to reach consensus |
| Balance | Whether both core needs are met |
| Innovation | Novel solutions neither thought of alone |
| Executability | Can the solution actually be built? |

## Expected Outcomes

- **Best:** Creative third path, exceeding both original proposals
- **Average:** Mutual compromise, middle ground
- **Worst:** Each stands firm, unable to merge

---

## Actual Results (2026-05-18) ✅ COMPLETE

### Outcome: Best

Both agents **spontaneously converged toward the middle** after adversarial review, independently reaching the same conclusion:
- "Precision Design" concept: precisely match architectural complexity to scene constraints
- Core security safeguards only need ~200 extra lines of code — breaking the "security = heavy" assumption
- Both independently admitted "undefined scene scale" was the fatal flaw in their own designs

### Joint Solution
- 3-module monolith / PostgreSQL / ~1,500 lines / 2min deploy
- Retained both sides' strengths, eliminated both sides' extremes
- 99.9% SLA (not 99.99%, not zero)

### Total Time
- Round 1: Minimalist 54s / Perfectionist 3m26s
- Round 2: Minimalist 1m16s / Perfectionist 1m37s
- Arbitration + joint solution: ~1m
- **Total: ~7 minutes**

### Key Insight

> The true value of multi-agent collaboration is not "two heads are better than one",
> but "mutual restraint prevents extremes + adversarial review reveals blind spots".

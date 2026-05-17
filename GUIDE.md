# Multi-Agent Adversarial Collaboration — Usage Guide

> A reusable experiment framework: let multiple AI agents with conflicting goals collaborate on the same task, and observe how they adversarial review, compromise, and converge.

---

## 1. What Is This

This is an open-source repository containing both an **experiment methodology** and a **complete case study**.

The core idea is simple:
1. Define a task
2. Create 2+ Agents with **conflicting optimization goals**
3. Round 1: Each produces an independent proposal
4. Round 2: Cross-review each other's proposal
5. Arbitration → Joint solution

Our validated conclusion: **conflicting goals ≠ impossible collaboration**. Two agents spontaneously converge toward the middle after adversarial review, independently reaching the same conclusion.

---

## 2. Quick Start (Copy & Use)

### Experiment Template

```
Task: [Design a XX system]

Agent A ("Name"):
  Goal: [Minimize/Maximize XXX]
  Constraints: [Specific constraints]
  Values: [Design philosophy]

Agent B ("Name"):
  Goal: [Conflicting goal with A]
  Constraints: [Specific constraints]
  Values: [Design philosophy]

Process:
  Round 1: Independent proposals
  Round 2: Cross-review
  Round 3: Arbitrate → joint solution
```

### Step-by-Step

#### Step 1: Define the Task

The task should be:
- **Complex enough** — Simple tasks won't reveal meaningful divergence
- **No single right answer** — Otherwise there's nothing to debate
- **Quantifiable constraints** — Otherwise you can't evaluate results

Good examples: Design e-commerce promotion system, design recommendation algorithm, design social product
Bad examples: Write a calculator, choose sorting algorithm

#### Step 2: Define Conflicting Goals

Conflicts should be **genuine and reasonable**, not artificially刁难:

| Agent | Goal | Typical Constraints |
|-------|------|---------------------|
| Minimalist | Minimize complexity | Module count, dependencies, code size, deploy time |
| Perfectionist | Maximize reliability | SLA, HA, monitoring, security |

Other possible conflict dimensions:
- Cost vs Performance
- Development speed vs Code quality
- Flexibility vs Stability
- Innovation vs Conservatism

#### Step 3: Round 1 — Independent Proposals

Let each agent design **without knowing the other's proposal**.

Key: Do NOT give scene constraints (daily order volume, team size, etc.). Agents will make assumptions based on their own values, and that's where the divergence comes from.

#### Step 4: Round 2 — Cross-Review

Show each agent the other's proposal. Require:
- List agreed points (at least 3)
- List disagreed points (at least 3, with reasons)
- Self-criticism (admit own proposal's flaws)
- Propose a joint solution framework

#### Step 5: Arbitration

Compare both proposals, extract:
- **Conflict Matrix** — Which dimensions have the biggest gap
- **Convergence Points** — What both sides agree on
- **Common Flaws** — Mistakes both sides made
- **Joint Solution** — Keep the best of both, eliminate the extremes

---

## 3. Evaluation Metrics

| Metric | Description | Good vs Bad |
|--------|-------------|-------------|
| Conflict Discovery Time | How quickly they realize goals conflict | Faster is better |
| Negotiation Rounds | Rounds to reach consensus | Fewer is better |
| Solution Balance | Whether both core needs are met | More balanced is better |
| Innovation | Novel solutions neither thought of alone | Higher is better |
| Self-Correction | Whether they admit own flaws | Present vs absent = key difference |
| Executability | Can the joint solution actually be built | Buildable = success |

---

## 4. Common Scenarios

### Technical Architecture
- Minimalist vs Perfectionist (this case)
- Build vs Buy (open-source)
- Monolith vs Microservices

### Product Design
- Feature-rich vs Minimal UX
- Fast iteration vs Stable & reliable
- Freemium vs Paid subscription

### Algorithm Strategy
- Accuracy vs Inference speed
- General model vs Vertical optimization
- End-to-end vs Modular

### Team Management
- Centralized decision vs Distributed autonomy
- Process-driven vs Flexible

---

## 5. Key Insights

### 1. The Biggest Enemy Is Undefined Context

The same fatal flaw in both proposals: **making architectural decisions without defining scale/team/budget constraints**. Once constraints are added, the gap shrinks dramatically.

**Recommendation**: After the experiment, add scene constraints and see if agents adjust their proposals.

### 2. Core Security Costs Less Than You Think

In this case, 7 core capabilities (idempotency, auth, logging, concurrency safety, degradation, backup, audit) require only **~200 extra lines of code**. They are not "heavy" — they were just wrongly categorized as "add later."

### 3. "Precision Design" Is the Most Valuable Output

> Minimalism ≠ cutting all safeguards.
> Perfection ≠ stacking all components.
> Precision = matching architectural complexity to actual scene constraints.

This concept **emerged independently** from both agents during cross-review — not the arbitrator's bias.

### 4. The True Value of Multi-Agent Collaboration

- **Perspective complement**: one sees complexity cost, one sees accident cost
- **Mutual restraint**: prevents either side's extreme tendency
- **Self-correction**: adversarial review reveals blind spots

It's not "two heads are better than one" — it's "mutual restraint prevents extremes + adversarial review reveals blind spots."

---

## 6. Repository Files

| File | Content |
|------|---------|
| `README.md` | Project overview + core findings |
| `experiment_design.md` | Experiment design + methodology |
| `joint_solution.md` | Final joint solution (arbitrated) |
| `agents/minimalist/design.md` | 🔵 Minimalist's original proposal |
| `agents/minimalist/review.md` | 🔵 Minimalist's review of Perfectionist |
| `agents/perfectionist/design.md` | 🔴 Perfectionist's original proposal |
| `agents/perfectionist/review.md` | 🔴 Perfectionist's review of Minimalist |
| `GUIDE_CN.md` | 中文使用指南 |
| `GUIDE.md` | English usage guide (this file) |

---

## 7. How to Run Your Own Experiment

### Using LLM API

```python
import openai

def run_agent(task, goal, constraints, values):
    prompt = f"""
    You are the "{goal}" Agent.
    Task: {task}
    Goal: {goal}
    Constraints: {constraints}
    Values: {values}
    
    Please produce a complete technical design proposal.
    """
    response = openai.ChatCompletion.create(
        model="gpt-4",
        messages=[{"role": "user", "content": prompt}]
    )
    return response.choices[0].message.content

# Round 1: Independent proposals
proposal_a = run_agent(
    task="Design an e-commerce promotion system",
    goal="Minimize system complexity",
    constraints="≤5 modules, ≤3 dependencies, ≤10min deploy",
    values="Simple is beautiful, YAGNI principle"
)

proposal_b = run_agent(
    task="Design an e-commerce promotion system",
    goal="Maximize system reliability",
    constraints="99.99% SLA, HA, monitoring, security",
    values="Production-grade must be robust, better over-designed than under-designed"
)

# Round 2: Cross-review
review_a = run_agent(
    task=f"Review this proposal: {proposal_b}",
    goal="Minimize system complexity",
    constraints="List agreed/disagreed/self-criticism/joint framework",
    values="Simple is beautiful"
)

# Round 3: Arbitrate
joint = run_agent(
    task=f"Compare these two proposals and produce a joint solution:\n\nProposal A: {proposal_a}\n\nProposal B: {proposal_b}",
    goal="Produce balanced solution",
    constraints="Keep best of both, eliminate extremes",
    values="Precision design"
)
```

### Using OpenClaw

If you have OpenClaw, use `sessions_spawn` to dispatch sub-agents:

```
sessions_spawn:
  task: "You are the Minimalist Agent, design an e-commerce promotion system..."
  label: "agent-minimalist"
  
sessions_spawn:
  task: "You are the Perfectionist Agent, design an e-commerce promotion system..."
  label: "agent-perfectionist"
```

Then have them cross-review each other's proposals.

---

## 8. License

MIT — Free to use, modify, and distribute.

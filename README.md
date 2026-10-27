# Google Ads Agent Staffing Planner

## Overview

Google Ads agent staffing is managed locally by country, with agents supporting advertisers in their respective markets.

Each productive agent can support up to **10 advertisers simultaneously**. However, advertiser demand fluctuates over time, creating a staffing challenge:

- Too many agents creates idle capacity and unnecessary salary cost.
- Too few agents increases advertiser waiting time.
- Advertisers that wait too long may never receive support.
- Insufficient support capacity may result in lost incremental advertising spend.
- Hiring and termination decisions cannot take effect immediately.

This project builds an **AI-assisted workforce planning application** that recommends monthly agent staffing decisions for each country in **2026**.

The solution combines:

- Python-based demand forecasting
- Deterministic staffing simulation
- Prompt engineering
- Structured LLM output
- LangGraph workflow orchestration

The GenAI component acts as a **staffing planning assistant**. It recommends hires and termination notices, while deterministic Python functions validate whether the recommendation actually satisfies the business rules.

---

# Business Problem

The goal is to determine the appropriate number of productive agents for each country and month in 2026 while balancing three objectives.

### Cost efficiency

Minimize unnecessary staffing and idle agent capacity while accounting for:

- Agent salaries
- Ramp-up costs
- Termination costs

### Timely advertiser support

Reduce:

- Advertiser waiting time
- Advertiser expiration
- Capacity shortages

while maximizing the number of eligible advertisers receiving support.

### Scalability

Adjust staffing as demand changes while respecting:

- One-month hiring ramp-up
- One-month termination notice
- 60-day advertiser support duration
- 60-day maximum advertiser waiting period

---

# Business Rules

## Advertiser eligibility

When an advertiser signs up, the advertiser has a projected annual advertising budget.

Each country has an eligibility threshold.

```text
Projected Annual Budget > Country Threshold
                    ↓
             Eligible advertiser
```

Only eligible advertisers are included in the staffing model.

---

## Agent capacity

Each productive agent can support:

```text
Maximum 10 advertisers
```

Capacity is therefore:

```text
Advertiser Capacity = Productive Agents × 10
```

---

## Advertiser lifecycle

Eligible advertisers follow the lifecycle below:

```text
Signup
  ↓
Eligible?
  ↓
Waiting Pool
  ↓
Capacity available?
 ┌─────────────┐
Yes            No
 ↓              ↓
Assigned       Wait
 ↓              ↓
60-day          More than
support         60 days?
 ↓              ↓
Graduated     Expired
```

Advertisers are assigned using **First-In, First-Out (FIFO)**.

---

## Agent lifecycle

### Hiring

An agent hired in month `M` requires one month of ramp-up.

For example:

```text
January
Hire 5 agents
     ↓
5 agents are ramping and paid
     ↓
February
5 agents become productive
```

### Termination

A termination requires one month's notice.

For example:

```text
March
Issue 3 termination notices
     ↓
Agents remain productive in March
     ↓
April
3 agents leave
```

Each termination incurs a cost equal to:

```text
40% × Annual Agent Salary
```

---

# Input Data

The project uses the following datasets.

## 1. Agent salary and eligibility thresholds

Example:

```text
Country | Eligibility_Threshold | Annual_Agent_Salary_USD
Brazil  | ...                   | ...
Japan   | ...                   | ...
```

Used for:

- Determining advertiser eligibility
- Calculating monthly agent salary
- Calculating termination costs

---

## 2. Advertiser sign-ups

Historical advertiser-level data for **2024 and 2025**.

Example:

```text
Advertiser_ID
Country
Sign_Up_Date
Projected_Annual_Budget
```

Used for:

- Eligibility calculation
- Historical demand analysis
- 2026 demand forecasting
- Forecast advertiser generation

---

## 3. Existing agent staffing

Example:

```text
Country | Existing_Agent_Count
Brazil  | 189
...
```

This represents the productive agent headcount available at the beginning of 2026.

All agents are assumed to start 2026 with no advertisers already assigned.

---

# Solution Architecture

```text
Refer to PowerPoint deck for flow and solution architect.
```

---

# Core Python Functions

## `build_staffing_schedule()`

Converts monthly staffing decisions into actual productive staffing and costs.

### Responsibilities

The function:

- Applies previous-month hiring
- Applies previous-month termination notices
- Calculates productive agents
- Tracks ramping agents
- Calculates paid headcount
- Calculates salary cost
- Calculates termination cost
- Calculates total staffing cost

---

## `simulate_country()`

Runs the advertiser and capacity simulation one day at a time.

### Daily process

For every day:

```text
1. Graduate completed advertisers
2. Add today's eligible advertiser sign-ups
3. Expire advertisers waiting longer than 60 days
4. Calculate available agent capacity
5. Assign advertisers FIFO
6. Record daily metrics
```

Possible status values include:

```text
NOT_ARRIVED
WAITING
ASSIGNED
GRADUATED
EXPIRED
```

---

## `evaluate_plan()`

Evaluates the outcome of a completed staffing simulation.

### Service metrics

```text
advertisers_supported
advertisers_expired
expiration_rate

average_wait_days
p90_wait_days
max_wait_days
```

### Capacity metrics

```text
average_utilization
peak_utilization
```

### Financial metrics

```text
total_staffing_cost
potential_revenue_uplift
expected_revenue_uplift
net_financial_impact
```

---

# Expected Advertising Spend Uplift

Agent support is modeled using the supplied uplift probability distribution:

```text
Ad Spend Uplift | Probability
0%              | 5%
5%              | 15%
10%             | 25%
15%             | 25%
20%             | 20%
25%             | 10%
```

The expected uplift is:

```text
0%  × 5%
+ 5%  × 15%
+ 10% × 25%
+ 15% × 25%
+ 20% × 20%
+ 25% × 10%

= 13.75%
```

The project uses this measure as the modeled financial benefit of advertiser support.

---

# Baseline Scenario

Before asking the AI assistant to change staffing, the project establishes a **no-intervention baseline**.

For every country:

```text
Hires = 0
Terminations = 0
```

Existing agent staffing therefore remains unchanged throughout 2026.

The baseline is passed through:

```text
build_staffing_schedule()
        ↓
simulate_country()
        ↓
evaluate_plan()
```

This answers:

> What happens during 2026 if the organization makes no staffing changes?

The baseline provides the benchmark against which AI recommendations are evaluated.

---

# Heuristic Plan

A heuristic plan is built but never shown to the LLM. It acts as a rule-based plan used only as fallback/benchmark.

For every country:
```text
Each advertiser is supported for about two months, so the load in month m is roughly this month's plus last month's eligible sign-ups (or the previous two months' at the start of the month, whichever is larger), plus a 10% buffer, divided by 10.
Hire one month early to cover ramp-up. January cannot change
Hold rule. Firing costs 40% of annual salary (4.8 months of pay) plus a ramp-up month if the agent is replaced later, so agents needed again within 6 months are kept.
```
The heuristic plan is passed through:

```text
build_staffing_schedule()
        ↓
simulate_country()
        ↓
evaluate_plan()
```

---

# GenAI Staffing Assistant

The GenAI assistant does **not** directly calculate staffing cost or advertiser outcomes.

The assistant recommends:

```text
Country
Month
Staffing_Status
Hires
Terminations
Reason
```

---

# Baseline vs Heuristic vs GenAI Comparison 

The final recommendations are made and passed through a write_memo node, which write out an analysis and justification of why a specific plan is chosen.

It includes key decisions, assumptions, and the impact on revenue and costs.


---

# Running the Project

Directly running the notebook.

---

# Important Modeling Assumptions

The current implementation assumes:

1. Each productive agent supports a maximum of 10 advertisers simultaneously.
2. Advertiser support lasts exactly 60 days.
3. Advertisers may wait up to 60 days.
4. Waiting advertisers are assigned FIFO.
5. All existing agents begin 2026 without existing advertiser assignments.
6. New agents require one month of ramp-up.
7. Ramping agents receive salary but provide zero capacity.
8. Termination requires one month's notice.
9. Agents remain productive during their notice month.
10. Termination costs 40% of annual salary.
11. The expected advertiser spend uplift used by the evaluation model is 13.75%.
12. Forecast growth is derived from 2024-2025 historical demand.

---

# Limitations

This project uses GenAI to make staffing recommendations, it should **not** be described as a mathematically proven global optimum.

Additional limitations include:

- Only two years of historical demand are available.
- Demand forecasting is sensitive to unexpected market changes.
- Synthetic 2026 advertiser records approximate within-month arrival patterns.
- Agent productivity is assumed to be uniform.
- Unexpected attrition, sickness, leave, and recruitment constraints are not currently modeled.
- A 2026-only evaluation may undervalue staffing consequences extending into 2027.
- Expected incremental advertising spend is a modeled business value and should not automatically be interpreted as accounting revenue.

---

# AI Engineering Techniques Demonstrated

## Prompt Engineering

The LLM receives explicit:

- Business rules
- Staffing constraints
- Historical demand
- Forecast demand
- Simulation results
- Capacity indicators
- Financial information
- Output instructions

## Structured Output

Pydantic schemas constrain the LLM response to validated monthly staffing decisions.

## LangGraph

LangGraph coordinates:

```text
Context
→ Plan
→ Simulate
→ Evaluate
→ Validate
→ Revise
→ Compare
```

This allows the AI assistant to propose a staffing decision while deterministic Python components verify whether the proposed decision actually works.

---

# Methodology Summary

The application does not ask an LLM to guess the required number of agents directly.

Instead:

```text
Historical data
      ↓
Forecasting
      ↓
Deterministic baseline / Heuristic Approach / GenAI recommendation
      ↓
Deterministic staffing lifecycle
      ↓
Daily advertiser simulation
      ↓
Performance evaluation
      ↓
Validation / revision
      ↓
Final recommendation
      ↓
Analysis memo written for manager review
```

This design combines the flexibility of GenAI with deterministic business-rule enforcement and quantitative validation.
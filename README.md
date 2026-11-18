# Check Less, Fail Less?
**Dependency-Boundary Validation versus Exhaustive Validation in Deterministic Tool-Using Agents**

[![Status](https://shields.io)](#status)
[![Compute](https://shields.io)](#compute--environment)

> **Note:** This repository contains the pre-registration-style experimental protocol, deterministic simulation environment specification, and model-derived projected results for *Check Less, Fail Less?*. **This is a proposed experiment, not a completed empirical run or novelty claim.**

---

## Overview

Tool-using agents rarely fail because tools crash (HTTP 500s or malformed JSON); they fail because tools return **plausible-but-incorrect (PBI)** outputs—well-formed, schema-valid, unit-consistent values that silently corrupt downstream execution. 

Rather than asking *whether* to validate tool outputs, this experiment asks **where** to place validators in a task Directed Acyclic Graph (DAG) to minimize final-task failures per unit of validation overhead. Using a deterministic, CPU-only simulation across **40 tasks**, **9 PBI fault classes**, and **20 paired fault schedules** (800 paired runs per arm; 3,200 total runs), we evaluate four validation policies under byte-identical fault schedules, validators, and repair budgets:

* **`P0` (Terminal Only):** No intermediate checks; validate only at the final output.
* **`P1-r` (Random Placement):** Validate at \(k\) randomly sampled nodes (matched in count to `P1`).
* **`P1` (Dependency-Boundary Validation):** Validate only at structurally selected boundaries (mean 2.6 checks per task out of 9.4 nodes).
* **`P2` (Exhaustive Validation):** Validate after every single tool call (mean 9.4 intermediate checks + terminal check).

---

## Key Projected Findings

*All figures below are derived from the explicit generative model in Appendix F prior to pilot execution.*

| Endpoint | `P0` (Terminal) | `P1-r` (Random) | `P1` (Boundary) | `P2` (Exhaustive) |
| :--- | :---: | :---: | :---: | :---: |
| **Final Task Success (FSR)** | 32.5% | 52.3% | **69.7%** | 74.0% |
| **Fault Detection Rate (TDR)** | 40.2% | 66.5% | 81.2% | **96.8%** |
| **Conditional Recovery (\(R_{\text{cond}}\))** | 10.0% | 55.0% | **85.0%** | 69.6% |
| **Absolute Faults Recovered** *(of 1,120)* | 45 | 410 | **773** | 754 |
| **Mean Propagation Depth** | 4.8 | 3.1 | 1.4 | **0.6** |
| **False Positives / Task** | 0.03 | 0.16 | **0.16** | 0.83 |
| **Validator Calls / Task** | 1.0 | 3.6 | **3.6** | 10.4 |
| **Cost Multiplier** | 1.00× | 1.18× | **1.24×** | 1.63× |

### Core Takeaways
1. **Efficiency Dominance:** Boundary validation (`P1`) captures **89.6%** of the achievable FSR gain over `P0` at **24%** of `P2`'s cost increment (**2.35×** higher cost-efficiency).
2. **Placement Over Quantity:** Randomly placing the exact same number of checks (`P1-r`) achieves only **52.3% FSR**—a **17.4 percentage-point** drop compared to `P1`.
3. **The Recovery Inversion:** Exhaustive checking (`P2`) detects more faults (1,084 vs. 909) but recovers **fewer faults in absolute terms** (754 vs. 773). With a shared repair budget of 2 attempts per run, `P2`'s **5.2× higher false-positive rate** (0.83 FP/task) exhausts the repair budget on false alarms before genuine faults can be fixed.

---

## Experimental Architecture

### 1. Task Suite (40 Tasks, 376 Nodes, 561 Edges)
Tasks are authored as typed artifact DAGs across four families (10 tasks each; 6–14 nodes; mean depth 6.1):
* **`F1` Extract–Transform–Aggregate:** Ingest, parse, normalize, aggregate, render.
* **`F2` Retrieve–Compute:** Query corpus, extract numerics, compute scalar, compare threshold.
* **`F3` Multi-Source Join:** High fan-in reconciliation across heterogeneous sources.
* **`F4` Orchestrate–Report:** Sequential service calls assembled into a structured report.

### 2. Boundary Selection Criteria (`B`)
A node \(v\) is selected for `P1` validation (capped at 5 boundaries per task, prioritized `B1 > B4 > B2 > B3`) if it meets any of:
* **`B1` Type-Contract Change:** Output artifact is coerced on an outgoing edge or crosses tool families.
* **`B2` Fan-In \(\ge 2\):** Node aggregates inputs from two or more upstream producers.
* **`B3` High Fan-Out (\(\ge 3\)):** Node output feeds three or more downstream consumers.
* **`B4` Irreversible / Expensive Sink:** Top-decile execution cost or marked side-effecting.
* **`B5` Terminal Acceptance:** Final task output node \(v^\star\) (included across all arms).

### 3. PBI Fault Catalogue (9 Classes)
All injected mutations preserve schema validity (`200 OK`, valid types). Across 1,120 injected faults per arm (1 or 2 faults per run), **60.0%** are ground-truth load-bearing:
* `unit_confusion` • `stale_source` • `silent_truncation` • `off_by_one` • `field_swap` • `entity_misresolution` • `rounding_drift` • `plausible_hallucination` • `implementation_bias`

### 4. Layered Checkers & Repair Escalation
* **6-Layer Validators:** Executed in order with early short-circuiting: *(1) Schema \(\rightarrow\) (2) Range \(\rightarrow\) (3) Unit \(\rightarrow\) (4) Cross-field invariant \(\rightarrow\) (5) Independent recomputation \(\rightarrow\) (6) Reference lookup*.
* **Repair Ladder (Budget = 2/run):** Triggered only on a validator flag, ordered by cost: `R1` Local re-derive (alternate tool implementation) \(\rightarrow\) `R2` Source re-fetch \(\rightarrow\) `R3` Upstream producer escalation.

---

## Practitioner Decision Rule

Based on the projected cost-benefit analysis:
1. **Default to Boundary Checking (`P1`):** Use criteria `{B1, B2, B5}` with a cap of **3 boundaries** per task (ablations show `B3` and `B4` contribute minimal marginal gain).
2. **Use `P2` only in high-stakes domains:** Choose exhaustive checking only when a ~4 percentage-point FSR gain justifies **2.6×** the validation spend.
3. **Avoid `P2` with noisy validators:** If per-check validator specificity drops below **~0.92**, false positives degrade `P2` far faster than `P1`.
4. **Increase repair budget before check count:** Raising the repair budget from 2 to 4 pushes `P1` to **76.4% FSR**—outperforming baseline `P2` (74.0%) at a fraction of the check overhead.

---

## Open Issues & Pre-Registration Roadmap

* **ERR Metric Standardization:** To eliminate denominator ambiguity in prior reviews, all evaluations will report six explicit Error-Recovery Rate variants: \(\text{ERR}_{\text{cond}} = R/D\), \(\text{ERR}_{\text{marg}} = R/F\), and \(\text{ERR}_{\text{pen}} = (R - 2\phi)/F\) across both all faults and load-bearing faults.
* **Related Work Positioning:** Before final empirical execution, additional paired arms will benchmark `P1` directly against **Sherlock** (`arXiv:2511.00330`), **AgentProp-Bench** (`arXiv:2604.16706`), and classical fail-stop dynamic programming checkpointing.

---

## Planned Repository Structure

```text
├── tasks/              # 40 deterministic task DAG specifications (YAML)
├── faults/             # 9 PBI fault mutators & seeded schedule generator (seeds 10000–10799)
├── checkers/           # 6-layer deterministic validator suite
├── policies/           # P0, P1-r, P1 (criterion B), and P2 execution & repair routines
├── model/              # Appendix F generative model & re-scaling scripts
└── notebooks/          # Pre-registered McNemar, Wilcoxon, and mixed-effects analysis
```

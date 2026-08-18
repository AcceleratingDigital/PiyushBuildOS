# Evolution to Gated Graph Architecture

This document defines the transition of the PiyushBuildOS from a strictly linear pipeline to a "Gated Graph" model. The goal is to introduce agentic agility (self-correction and recursive decomposition) without sacrificing the S-S-D traceability and human-in-the-loop guardrails.

## 1. The "Gated Graph" Concept
Instead of a monolithic sequence (A $\rightarrow$ B $\rightarrow$ C), the process is evolved into a series of **Self-Correcting Clusters** separated by **Human-Verified Gates**.

### The Macro-Pipeline (Remains Linear)
`Research` $\rightarrow$ `Spec/Design` $\rightarrow$ `Build Cluster` $\rightarrow$ `Formal QA/Review` $\rightarrow$ `Ship`

### The Build Cluster (The Micro-Graph)
The "Build" stage is transformed from a single agent action into a dynamic loop:
1. **Dispatch:** Coordinator triggers a Build Cluster for a specific `feature/{slug}`.
2. **Internal Loop:** 
    - **Coder Node:** Implements a sub-slice of the spec.
    - **Verifier Node (Mini-QA):** Runs a targeted test/check.
    - **Router Node:** If `FAIL` $\rightarrow$ loop back to Coder with error. If `PASS` $\rightarrow$ move to next sub-slice.
3. **Convergence:** Once all sub-slices are verified internally, the Cluster "promotes" the work to the formal QA gate.

---

## 2. Key Advanced Capabilities

### A. Recursive Task Decomposition
The Coordinator no longer just assigns a task; it can now dynamically decompose a "Large" scope task into a DAG (Directed Acyclic Graph) of smaller, interdependent tasks:
- **Parent Task:** "Build Finance Vertical"
- **Dynamic Sub-tasks:** [Import Engine] $\rightarrow$ [Categorizer] $\rightarrow$ [Anomaly Detector].
- **S-S-D Mapping:** Every node in the graph must still map to a specific Asana GID to maintain the "Intent $\rightarrow$ Reality" link.

### B. Self-Correction Loops (The "Inner Monologue")
To reduce the number of macro-reworks, agents use an internal "Critic" loop:
- **Draft $\rightarrow$ Critic $\rightarrow$ Refine $\rightarrow$ Output.**
- This prevents the "hallucination of success" by forcing the agent to find its own flaws before the human or the formal QA agent ever sees the code.

### C. State-Persistence (Graph Memory)
The "Build Cluster" maintains a shared **Working Memory** (separate from the long-term Knowledge Base) that tracks:
- **Hypotheses:** "I suspect the bug is in the TCC entitlement."
- **Attempt Log:** "Tried X $\rightarrow$ Failed with Y. Tried Z $\rightarrow$ Partial success."
- **Convergence State:** A record of which parts of the spec are "internally verified."

---

## 3. The "Safe Graph" Guardrails (The Non-Negotiables)

To prevent the "Agentic Chaos" typical of fully autonomous graphs, these constraints are mandatory:

1. **The Atomic PR Rule:** Regardless of how many micro-loops occur internally, the result of a Build Cluster must be a **single atomic PR** to `main`.
2. **The Circuit Breaker:** Every internal loop has a "Max Iteration" limit (e.g., 5 attempts). If convergence is not reached, the agent must **Freeze** and escalate to a Human with a "Failure Analysis Report."
3. **S-S-D Anchor:** The "Graph State" is temporary. The final truth must always be committed back to the S-S-D surfaces (Asana notes updated, Git commit pushed, Doc page generated).
4. **Human Sign-off:** No amount of internal "Agentic Convergence" replaces the **Human Final Seal** on a clean machine.

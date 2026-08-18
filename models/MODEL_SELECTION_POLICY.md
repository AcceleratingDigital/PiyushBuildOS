# LLM Selection Policy & Model Router Logic

This document defines the "Tiered Intelligence" approach for the agentic build process. To balance cost, speed, and reasoning depth, tasks are assigned to models based on their specific strengths.

## Model-to-Role Mapping

| Role | Primary Model | Fallback/Escalation | Reasoning |
|---|---|---|---|
| **Coordination / Architecture** | `algolia/xlarge` | `claude-sonnet-4-6` | Highest reasoning for triage and sequencing. |
| **Core Coding / Feature Build** | `algolia/xlarge` | `claude-sonnet-4-6` | High logic capability with local cost. |
| **QA Verification** | `algolia/medium` | `algolia/xlarge` | Speed for checklist verification. |
| **Tech Docs / Design** | `claude-sonnet-4-6` | `algolia/xlarge` | Superior HTML/CSS and structured documentation. |
| **Deep Security Review** | `gpt-5.6-sol` | `claude-sonnet-4-6` | Model diversity catches family-specific blind spots. |
| **General Interaction** | `algolia/medium` | `algolia/xlarge` | Balanced speed and accuracy. |

## The "Escalation" Rule
1. **Start Local:** Always attempt the task with the primary `algolia` model.
2. **Verify:** If the output passes the gate (QA or Review), do NOT retry with a more expensive model.
3. **Escalate on Failure:** Only move to Claude or OpenAI if:
    - The agent reports a "blocking" inability to solve the logic.
    - The QA/Review loop fails 3 times with the same error.
    - The task is explicitly flagged as "High Complexity" in the Build Readiness metadata.

## Model Router Architecture (S-S-D)
- **S-S-D Principle:** Feature specs must NOT name the model. The `Model Router` (a server-side component) makes the decision at runtime based on this policy.
- **Fallback Chain:** If the primary provider (e.g., LiteLLM) is unreachable, the router falls back to the next best model in the chain without interrupting the agent.

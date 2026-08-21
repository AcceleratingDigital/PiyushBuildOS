# Tool & Model Mapping Matrix

This document defines the exact technical stack used for each agent role. To maintain "Model Diversity" and avoid systemic blind spots, different tools and models are used across the pipeline.

## Agent Stack Mapping

| Agent Role | Tool/CLI | Primary Model | Fallback/Escalation | Key Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **Coordinator** | Hermes Agent | `algolia/xlarge` | `claude-sonnet-4-6` | Orchestration, Triage, Merging |
| **Coder** | `claude` CLI | `algolia/xlarge` | `claude-sonnet-4-6` | Feature Implementation |
| **QA** | `opencode` CLI | `algolia/medium` | `algolia/xlarge` | Spec Verification, Test Execution |
| **Reviewer** | `codex` CLI | `gpt-5.6-sol` | `claude-sonnet-4-6` | Security & Architecture Audit |
| **Tech Docs** | `claude` CLI | `claude-sonnet-4-6` | `algolia/xlarge` | Structured Markdown, Architecture Docs |
| **Marketing Site**| `claude` CLI | `claude-sonnet-4-6` | `algolia/xlarge` | HTML/CSS, Visual Design, Copy |

## Technical Implementation Details

### 1. The Proxy Layer (LiteLLM)
All CLI tools are routed through the LiteLLM proxy at `10.1.2.13:4000`. This allows the system to:
- **Mimic APIs:** The `claude` CLI believes it is talking to Anthropic, but LiteLLM routes it to a local `algolia` model.
- **Transparent Fallback:** If a primary model is offline, LiteLLM can route to a fallback without the agent crashing.

### 2. Execution Patterns
- **The Coder (`claude`):** Launched with `--dangerously-skip-permissions` and `ANTHROPIC_BASE_URL` pointing to the proxy.
- **The QA (`opencode`):** Launched with `-m litellm/<model>` to specify the verification model.
- **The Reviewer (`codex`):** Launched with `--skip-git-repo-check` for deep analysis of specific diffs.

## Why Diversify Tools?
We use a "Multi-Tool Strategy" to prevent a single-point-of-failure in reasoning:
- **Coder (Claude CLI)** is optimized for *generating* working code.
- **QA (OpenCode)** is optimized for *proving* that code fails or passes.
- **Reviewer (Codex)** is optimized for *auditing* code for non-functional flaws (security, leaks, complexity).

By using different tools, we ensure a "checks and balances" system where the agent that wrote the code is not the one verifying it, and the tool verifying it isn't the same one that reviewed it.

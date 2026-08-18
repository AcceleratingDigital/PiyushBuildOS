# Evaluation & Verification Process

This document outlines how we verify that the system is actually improving and that the agentic process is producing high-quality results.

## 1. The Dual-LLM Verification Pattern
To prevent "Model Echo Chambers" (where a model approves its own mistakes), we enforce cross-model validation:
- **Generation $\rightarrow$ Validation:** The model that generates a spec or a doc page must NOT be the model that validates it.
- **Code-Grounded Truth:** Validation agents must read the *actual source code* and compare it to the claims in the document, not just search for keywords.

## 2. The Rework Loop (Iterative Evaluation)
We treat "Needs Rework" as a primary evaluation metric:
- **Loop Depth:** If a feature requires >3 rework loops between Coder $\rightarrow$ QA $\rightarrow$ Reviewer, it is flagged for a "Design Audit."
- **Feedback Loop:** Feedback is captured in `docs/task-feedback/<slug>.md`, which serves as a training set for improving future prompts.

## 3. Success Metrics (S-S-D)
Success is measured by the state of the three truth sources:
- **Intent (Asana):** Does the delivered feature match the "WHAT" section of the spec?
- **Build (Git):** Does the code merge clean without regression?
- **Truth (Site):** Is the user-facing documentation accurate and live?

## 4. Performance Baseline
- **TCC/Permission Test:** A binary Pass/Fail on whether the app can successfully request and receive permissions on a clean install.
- **S-S-D Sync:** Verification that a change in the `main` branch is reflected on the marketing site within the specified sync window (15m).

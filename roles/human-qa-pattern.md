# Human QA & Beta Validation Pattern

This document defines the "Final Seal" process—how we move from a technically "shipped" feature to a "validated" user experience. 

## The Philosophy
Automated QA (OpenCode) and Agentic Reviews (Codex) verify that the code *meets the spec*. Human QA verifies that the *spec actually solves the user's problem* and that the *installation/onboarding* doesn't fail in the real world.

## The Validation Pipeline

### 1. The Clean-Install Baseline
Before testing any feature, the tester must perform a "Clean Slate" setup to ensure no stale state or cached bundles are biasing the results:
- Fresh `git pull` of `main`.
- Full re-package of the release binary.
- Complete wipe of `/Applications/hOS Server.app`.
- Wipe of `~/Library/Application Support/hOS/Skills/` (removes stale bundles).
- TCC Reset: `tccutil reset All <BundleID>` to ensure the permission flow is tested.

### 2. Vertical Slice Testing
Instead of testing a list of features, Human QA focuses on "Verticals" (complete user journeys):
- **The Mail Vertical:** Triage $\rightarrow$ Summarization $\rightarrow$ Draft Reply $\rightarrow$ Open Loop $\rightarrow$ Approval.
- **The Finance Vertical:** Import $\rightarrow$ Categorization $\rightarrow$ Anomaly Detection $\rightarrow$ Daily Brief.
- **The Family Vertical:** Chore Assignment $\rightarrow$ Kid Completion $\rightarrow$ Parent Approval $\rightarrow$ Reward.

### 3. The "Failure Mode" Mindset
Human testers are instructed **not to work around bugs**. 
- If a step fails, the tester stops and reports exactly what they saw. 
- The "Workaround" is a failure of the product; the "Finding" is the value.

## The Human QA Plan Template
Every major release must have a `beta-qa-plan.md` containing:
1. **Pre-flight Checklist:** Step-by-step clean install and admin config.
2. **Phased Test Cases:** 
    - **Phase 1: Core Platform** (Inference, Storage, Vault, Approvals).
    - **Phase 2: Vertical Features** (The primary value props).
    - **Phase 3: Admin & Ops** (Audit logs, Member mgmt, launchd).
    - **Phase 4: Security** (SQLi tests, Data isolation).
    - **Phase 5: Edge Cases** (Offline mode, Large data, Special characters).
3. **Results Summary Table:** A simple Pass/Fail matrix for every "Must-Have" item.

## Closing the Loop
- **Pass:** Feature is marked `status-released` (Final).
- **Fail:** Create a bug task in Asana $\rightarrow$ Tag `status-blocked` $\rightarrow$ Route back to Coder agent.

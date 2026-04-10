# Design Pillars - Debate Arena v2.0

## Core Pillars

-   **Recipe-card WORKFLOW**: `WORKFLOW.md` là sequential task list. Logic chi tiết nằm trong từng `AGENT_*.md`.
-   **Filesystem-as-state**: Agents communicate qua Markdown files trong `debates/[id]/workspace/`.
-   **Parallel independence**: PRO/CON và 3 ensemble judges chạy parallel sub-agents qua Task tool, không share context.
-   **Persona diversity**: 12-persona library + framework distance matrix ≥ 6 → epistemic diversity thật.
-   **Verification chain**: `verified_facts[]` mandatory + Tier A/B/C/D + FactCheck independent search.
-   **Ensemble judging**: 3 framework judges (U/R/P) blind + Meta-Judge aggregate + sealed mapping.
-   **Automation default**: Step 0 → 10 chạy liền mạch. Chỉ STOP khi escalation thật.
-   **AutoTest harness**: Karpathy-style bounded loops để tự tối ưu prompts qua các cycle.

---

## Version Tracking

| Version | Date | Author | Description |
|:---|:---|:---|:---|
| v1.0 | 2026-04-10 | Antigravity | Initial transcription from dashboard.jpg |

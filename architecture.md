# 🏗️ Pipeline Architecture

This document describes the high‑level design and flow of the Jenkins pipeline.

---

## 📊 Pipeline Flow Diagram

```mermaid
graph TD
    A[Developer pushes to Git] --> B[Jenkins Pipeline triggered]
    B --> C{Parameter: ROLLBACK?}
    C -->|True| Z[Rollback Stage: undo Kubernetes deployment]
    C -->|False| D[Checkout + Build Application]
    D --> E[SonarQube Analysis]
    E --> F{Quality Gate Passed?}
    F -->|No| G[❌ Pipeline Fails]
    F -->|Yes| H[Build Docker Image]
    H --> I[Trivy Vulnerability Scan]
    I --> J{Critical Vulnerabilities?}
    J -->|Yes| G
    J -->|No| K[Push to Docker Registry]
    K --> L{SKIP_CANARY?}
    L -->|Yes| Q
    L -->|No| M[Deploy Canary (1 replica)]
    M --> N[Canary Health Checks]
    N --> O{Healthy during Bake Period?}
    O -->|No| P[Delete Canary, Pipeline Fails]
    O -->|Yes| Q[Deploy Stable (2 replicas)]
    Q --> R[Promote: delete canary]
    R --> S[OWASP ZAP DAST Scan]
    S --> T[Application Info & Archive Artifacts]
    T --> U[✅ Pipeline Success]

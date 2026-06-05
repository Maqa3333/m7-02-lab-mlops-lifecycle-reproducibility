# ETA Model — MLOps Lifecycle

```mermaid
flowchart TD
    A["📦 Raw Data Lake\n(S3 partitioned by date)"]
    B["🔄 Feature Pipeline\nArtifact: dataset_hash SHA-256\n(e.g. sha256:a3f9c1...)"]
    C["🧪 Experiment Runs\nArtifact: mlflow run_id\n(e.g. run/7e2b4f)"]
    D["🏋️ Training Job\nArtifact: model weights\n+ training_config.yaml"]
    E["📊 Evaluation\nArtifact: eval_report.json\n(MAE, p95 latency, per-city slices)"]
    F{"Metrics pass\nauto-gate?"}
    G["🗂️ Registry: STAGING\nArtifact: model URI\n(models:/eta/staging#42)"]
    H["🔍 Human Review\n(ML Lead + Product Owner)"]
    I["🗂️ Registry: PRODUCTION\nArtifact: model URI\n(models:/eta/production#42)"]
    J["🚀 Canary Deployment\nArtifact: deployed_version tag\n(eta-v42-canary, 5% traffic)"]
    K["🌐 Full Production\nArtifact: deployed_version tag\n(eta-v42-prod, 100% traffic)"]
    L["📡 Monitoring\nArtifacts: drift_signal.json,\np95_latency_ms, MAE rolling-7d"]
    M["🗂️ Registry: ARCHIVED\nArtifact: model URI\n(models:/eta/archived#41)"]

    A -->|"⚙️ AUTO: DVC pipeline trigger\nArtifact: versioned dataset snapshot"| B
    B -->|"⚙️ AUTO: dataset_hash logged to MLflow"| C
    C -->|"⚙️ AUTO: best run selected by val MAE"| D
    D -->|"⚙️ AUTO: run_id + dataset_hash attached to artifact"| E
    E --> F
    F -->|"✅ MAE ≤ 3.5 min AND\np95 latency ≤ 150 ms\n⚙️ AUTO: push to Staging"| G
    F -->|"❌ Fails gate → alert on-call\n⚙️ AUTO: reject + log"| C
    G -->|"👤 MANUAL: ML Lead reviews\neval_report.json slice breakdown"| H
    H -->|"✅ Approved: per-city MAE ≤ 5 min\n+ no regression on late_delivery slice\n👤 MANUAL approval"| I
    H -->|"❌ Rejected → back to experimentation"| C
    I -->|"⚙️ AUTO: CD pipeline triggers\ncanary at 5% traffic"| J
    J -->|"⚙️ AUTO: promote if p95 error rate\n≤ baseline + 0.5% over 1 hour"| K
    K -->|"⚙️ AUTO: metrics collected every 5 min\nArtifact: drift_signal.json"| L
    L -->|"⚙️ AUTO: if PSI > 0.2 on any feature\nOR rolling-7d MAE > 4.5 min\n→ trigger retraining"| B
    L -->|"⚙️ AUTO: previous production version\nmoved to Archived on new promotion"| M

    style G fill:#ffe0b2,stroke:#e65100
    style I fill:#c8e6c9,stroke:#2e7d32
    style M fill:#eeeeee,stroke:#757575
    style F fill:#e3f2fd,stroke:#1565c0
```

## Artifact Glossary

| Artifact | Format | Where Stored |
|---|---|---|
| `dataset_hash` | SHA-256 string | MLflow params + DVC `.dvc` file |
| `run_id` | UUID (MLflow) | MLflow Tracking Server |
| `training_config.yaml` | YAML | Git commit (pinned to run_id) |
| `eval_report.json` | JSON | MLflow artifacts |
| `model URI` | `models:/eta/<stage>#<version>` | MLflow Model Registry |
| `deployed_version` | Git tag `eta-v{N}-{canary\|prod}` | Git + K8s label |
| `drift_signal.json` | JSON (PSI per feature, rolling MAE) | Monitoring store (EvidentlyAI export) |

## Transition Legend

- **⚙️ AUTO** — triggered by CI/CD pipeline or monitoring hook, no human required  
- **👤 MANUAL** — requires explicit approval in the registry UI or PR review

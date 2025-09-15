# Customer Churn Prediction Platform (Databricks Lakehouse)

## 1) Problem

Subscription/commerce businesses lose revenue when customers silently stop using or cancel services. Retention teams react late because insights are batch-only, siloed, or not actionable in-product.

**Pain points**

* No single source of truth across product usage, transactions, and support.
* Static reports; no real-time intervention.
* Models aren’t traceable (no lineage/metrics/versioning).
* Manual ops; broken pipelines; unclear ownership and SLAs.

## 2) What We’re Trying to Achieve (Outcomes & KPIs)

* **Predict churn risk** per customer daily + near-real-time for active sessions.
* **Trigger timely actions** (offers/tickets/notifications) with clear decision thresholds.
* **Unify batch + streaming** on a **Delta Lakehouse** with governance (Unity Catalog).
* **Make impact measurable** with business KPIs:

**North-star KPIs**

* ↓ Churn rate by **5–10%** in pilot segment within 8–12 weeks.
* ↑ Retention campaign **conversion +3–7%**.
* Data platform KPIs: **>99% job SLA**; feature freshness **≤30 min** for streaming; **AUC ≥0.80** (target 0.85).

## 3) Our Solution (High-Level)

* Land all data to **Delta Lake** (bronze/silver/gold).
* Use **Autoloader** for batch files and **Kafka** for events into **Delta Live Tables** with data expectations.
* Build curated churn features in **Gold**; catalog with **Unity Catalog**.
* Train + track models via **MLflow**; register best model in **Model Registry**.
* Serve predictions via **Databricks Model Serving** (real-time API) + scheduled **batch scoring** jobs.
* Visualize churn drivers & segment risk with **Databricks SQL** dashboards.
* Orchestrate with **Databricks Workflows**; ship changes via **GitHub Actions CI/CD**.

## 4) Architecture

```
                         ┌──────────────────────────────────────────┐
                         │              Source Systems              │
                         │  OLTP DB (dim_customer), Payments API,   │
                         │  Support/CRM, Web/App events, CSV/Parquet│
                         └───────────────┬───────────────┬──────────┘
                                         │               │
                           Batch (Files) │               │ Streaming (Events)
                                         │               │
                                         ▼               ▼
                              ┌────────────────┐   ┌────────────────────┐
                              │  Autoloader    │   │   Kafka Connector  │
                              │  (Auto-ingest) │   │ (Structured Stream)│
                              └───────┬────────┘   └─────────┬─────────┘
                                      ▼                      ▼
                              ┌─────────────────────────────────────────┐
                              │      Delta Live Tables (Bronze)         │
                              │  Raw landing + schema evolution +        │
                              │  Expectations (DQ) + lineage             │
                              └───────────────┬─────────────────────────┘
                                              ▼
                              ┌─────────────────────────────────────────┐
                              │  DLT (Silver) - Clean/Conform/Dedupe    │
                              │  Standardize IDs, timezones, nulls      │
                              └───────────────┬─────────────────────────┘
                                              ▼
                              ┌─────────────────────────────────────────┐
                              │   Gold (Feature & Mart Layers)          │
                              │  churn_features, churn_labels, marts    │
                              └───────────────┬───────────┬────────────┘
                                              │           │
                                              │           │
                      ┌────────────────────────┘           └────────────────────────┐
                      ▼                                                           ▼
            ┌───────────────────────────┐                               ┌───────────────────────────┐
            │  ML (PySpark/MLflow)      │                               │  Analytics (DB SQL)       │
            │ Train/track/compare runs  │                               │ Dashboards & alerts       │
            │ Register model (Registry) │                               │ Self-service exploration  │
            └─────────────┬─────────────┘                               └───────────┬──────────────┘
                          ▼                                                            ▼
                 ┌──────────────────────┐                                   ┌──────────────────────┐
                 │  Serving & Scoring   │                                   │  Orchestration/CI/CD │
                 │ Real-time API (MS)   │◄───────────Feature lookup────────►│ Workflows + Actions  │
                 │ Batch jobs (Gold)    │                                   │ Tests/Deploy/Promote │
                 └──────────────────────┘                                   └──────────────────────┘

Governance/Observability across all layers: Unity Catalog (RBAC/lineage), Audit, Cost controls, Alerts
```

## 5) Data Sources (doc-friendly enumeration)

**Core (required)**

1. **Customer master (batch)** — profile, demographics, signup date, plan, status.
2. **Transactions/Payments (batch)** — invoices, amounts, success/failure, refunds.
3. **Product usage (stream + batch)** — logins, sessions, pageviews, feature toggles.
4. **Support/CRM tickets (batch)** — ticket count, categories, first-response time, CSAT.
5. **Marketing interactions (batch)** — email/SMS campaigns, opens/clicks, offers redeemed.

**Enrichment (optional)**

* **Holiday calendar**, **geo/demographic bins**, **pricing/plan catalog**, **A/B test flags**.

> For a public, reproducible build you can:
>
> * Use **IBM Telco Churn** (for labels + structure) to seed schemas.
> * Generate synthetic events for streaming via Kafka (customer activity).
> * Produce daily CSV/Parquet drops for payments/support.

## 6) How We Do It (Step-by-step Build)

### A. Governance & Project Scaffolding

* Enable **Unity Catalog**; create **metastore**, **catalog** (`churn`), **schemas**: `bronze/silver/gold/mlops`.
* Set up **repos** (Git integration), **branching** (`main`, `dev`), **Secrets** (tokens/keys).
* Define **data contracts** (table schemas + SLAs + expectations).

### B. Ingestion

* **Autoloader** to watch `/raw/{domain}/` for CSV/JSON/Parquet (schema inference + evolution).
* **Kafka** → **Structured Streaming** for events (`topic: customer.events`), checkpointing to `/checkpoints/`.
* Land to **Bronze** delta tables with minimal transforms, add **DLT expectations** (type/nonnull/range).

### C. Transform & Quality (DLT → Silver)

* Clean & conform keys (customer\_id), standardize timestamps/timezones, dedupe by natural keys.
* Join across domains to create consistent **entity views** (customer, transactions, support, marketing).
* Add data tests (e.g., “no orphan transactions”, “event ts monotonic within session”).

### D. Feature Engineering (Gold)

* **Recency/Frequency/Monetary**: `days_since_last_use`, `sessions_7/30d`, `avg_order_value`, `mrr`.
* **Engagement**: feature adoption counts, streaks, time-on-core-features.
* **Support burden**: tickets\_30d, last\_ticket\_category, CSAT trend.
* **Promotions**: recent\_offer\_exposure, redemption.
* **Labels**: churn = inactive N days or canceled flag.
* Write `gold.churn_features` and `gold.churn_labels` with **feature lookup keys**.

### E. Modeling (PySpark + MLflow)

* Train baselines: **Logistic Regression**, **XGBoost**.
* Use **time-based splits**; evaluate **AUC/PR-AUC**, **Calibrated precision\@K**.
* **MLflow**: log params/metrics/artifacts, compare runs, register best model (`churn_model:staging → production`).
* SHAP/feature importance for explainability.

### F. Serving & Scoring

* **Real-time**: enable **Databricks Model Serving** endpoint, perform **feature lookup** from Gold/online store.
* **Batch**: daily job writes `gold.churn_scores_daily` with risk bands and recommended action.
* Output **actionable feed** (`risk_band`, `next_best_action`) for CRM/MarTech.

### G. Analytics & BI

* **Databricks SQL**:

  * *Churn Overview*: rate by segment/plan/region; trend lines.
  * *Driver Analysis*: partial dependence/feature importance summaries.
  * *Campaign Performance*: uplift vs control for interventions.
* Set alerts on churn spikes or score distribution drift.

### H. Orchestration & CI/CD

* **Databricks Workflows** DAG:

  1. Ingest batch → 2) Stream jobs up → 3) DLT silver → 4) Gold feature build → 5) Train (nightly/weekly) → 6) Batch scoring → 7) Refresh dashboards.
* **GitHub Actions**: run unit tests (PyTest/dbt tests), style checks, deploy notebooks/jobs via Databricks CLI; promote MLflow model on metric gates.

### I. Observability & Cost

* Monitor **job durations/retries**, **cluster costs**, **storage growth** (optimize with file compaction, `OPTIMIZE ZORDER`).
* Track **model/data drift**; alert if PSI/KS exceeds threshold.

## 7) Data Model (Working Tables)

* `bronze.*` raw ingests per domain
* `silver.customer`, `silver.transactions`, `silver.events`, `silver.support`, `silver.marketing`
* `gold.churn_features` (one row per customer per day)
* `gold.churn_labels` (supervised labels)
* `gold.churn_scores_daily` (score, risk band, action)
* `gold.marts_*` (dashboards)

## 8) Acceptance Criteria (Definition of Done)

* End-to-end pipeline runs via **Workflows** with **>99% on-time** last 7 days.
* **Streaming** events processed with **<30 min** end-to-end latency.
* **Model AUC ≥0.80**, calibrated risk bands with business-approved thresholds.
* **Dashboards published** in Databricks SQL with documented metrics.
* **Runbook** (on-call), **README**, **Architecture diagram**, and **demo video** committed to repo.
* **Security**: tables governed under Unity Catalog; PII columns masked/ACL’d.

## 9) Tech Stack (Databricks-first)

* **Storage/Format**: Delta Lake
* **ETL/ELT**: Autoloader, **Delta Live Tables**, PySpark/SQL
* **Streaming**: Kafka + Structured Streaming
* **ML**: MLflow (tracking & registry), PySpark ML / XGBoost
* **Serving**: Databricks Model Serving (REST)
* **Orchestration**: Databricks Workflows
* **Governance**: Unity Catalog (RBAC, lineage)
* **CI/CD**: GitHub Actions + Databricks CLI/REST
* **BI**: Databricks SQL (and optional Power BI)

## 10) Repo Structure

```
customer-churn-databricks/
├─ notebooks/
│  ├─ 01_bronze_autoloader.py
│  ├─ 02_silver_dlt.sql          # or Python DLT pipeline
│  ├─ 03_gold_features.py
│  ├─ 04_labels_build.sql
│  ├─ 05_ml_train_mlflow.py
│  ├─ 06_batch_scoring.py
│  └─ 07_sql_dashboards.sql
├─ dlt_pipelines/
│  └─ churn_dlt_pipeline.json    # DLT config (expectations, clusters)
├─ workflows/
│  └─ churn_workflow.json        # Databricks Workflows DAG
├─ tests/
│  ├─ test_expectations.py
│  └─ data_quality.yml
├─ infra/
│  └─ terraform/                 # (optional) UC/catalog/schema/permissions
├─ docs/
│  ├─ architecture.ascii
│  └─ runbook.md
└─ README.md
```

## 11) Demo Script (5–7 minutes)

1. Open DB SQL dashboard → show churn trend + top drivers.
2. Click through a customer profile with high risk → show feature breakdown.
3. Trigger manual inference via Serving endpoint (curl / notebook) → show response.
4. Show MLflow runs → model registry → staging→production promotion gate.
5. Show Workflows with green runs; open one notebook to display Autoloader/stream metrics.

---

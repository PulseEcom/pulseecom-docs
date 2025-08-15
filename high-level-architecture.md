# PulseEcom – High Level Architecture

## Overview
PulseEcom is a real-time e-commerce anomaly detection and insights platform.  
It ingests transactional and behavioral data from online retail systems, detects unusual patterns using machine learning and rules, and notifies stakeholders instantly.

The architecture is designed to:
- Operate in **real time** (low-latency stream + batch support).
- Use a **global model + calibration** that is **retrained weekly**.
- Be **extensible** for new signals, models, and destinations.
- Scale horizontally for high event throughput.

---

## 1. System Architecture (End-to-End)

```mermaid
flowchart LR
  subgraph Client["Client Systems"]
    EP[Event Producers<br/>(orders, clicks, returns)]
    WH[Webhook Receiver<br/>(client endpoint)]
  end

  EP -->|REST/Batch/Stream| GW[API Gateway]
  GW --> IN[Ingestion Service]
  IN -->|events.raw| K1[(Kafka/Redpanda)]
  K1 --> EN[Enrichment Service]
  EN -->|events.enriched| K2[(Kafka)]
  K2 --> SC[Scoring Service]

  SC -->|features| MS[Model Serving<br/>(global, weekly retrained)]
  MS -->|score(s)| SC

  SC --> DB[(Postgres)]
  SC -->|events.scored| K3[(Kafka)]
  K3 --> AL[Alerting/Webhooks]
  AL -->|signed POST| WH

  DB --> QS[Query & Metrics API]
  DB --> AI[AI Insights Service]
  AI --> QS
```
**Flow Summary**
1. Client systems send events via API Gateway.  
2. Ingestion service pushes raw events into Kafka.  
3. Enrichment service adds contextual metadata (product/category, geo-IP, device).  
4. Scoring service calls **Model Serving** for anomaly scores.  
5. Scored events are persisted in Postgres.  
6. Alerts are published to the client’s webhook endpoint.  
7. Query API and AI Insights provide historical and trend data.

---

## 2. Model Lifecycle with Weekly Retraining

```mermaid
flowchart LR
  H[Historical Scored Events<br/>(last N weeks)] --> FE[Feature Build & Filtering<br/>(exclude confirmed anomalies)]
  FE --> T[Retraining Job<br/>(weekly schedule)]
  T -->|calibration params| CAL[Calibration (thresholds, scaling)]
  T -->|global model artifact| GM[Model Artifact<br/>(e.g., IsolationForest)]
  CAL --> MS[Model Serving]
  GM --> MS
  subgraph Serving["Model Serving"]
    MS
    note right of MS: Hosts global_model@vN<br/>(cold start uses v0)
  end
  MS --> SC[Scoring Service]
  SC --> H
```
**Notes**
- **Cold Start**: The platform serves `global_model@v0` (pre-trained on synthetic + generic e-commerce data) on day 1.  
- **Weekly Retraining**:
  - Produces updated **calibration parameters** (thresholds, scaling).
  - Produces a new **global model artifact** if improvements are found.  
- **Model Serving** supports hot-swapping models without downtime and logs model version with each score.

---

## 3. Scoring + Calibration Decision Flow

```mermaid
flowchart LR
  E[Enriched Event] --> B1[Global/Base Model<br/>score_base]
  E --> R1[Rules/Stats<br/>(e.g., z-score, guards)<br/>score_rules]

  B1 --> N1[Normalize]
  R1 --> N2[Normalize]

  subgraph CAL["Calibration Layer (global)"]
    N1 --> W[Weighted Combiner]
    N2 --> W
    W --> TH[Compare to Threshold<br/>(derived weekly)]
  end

  TH -->|>= threshold| A[Anomaly = TRUE]
  TH -->|< threshold| N[Anomaly = FALSE]
```
**Calibration Layer Responsibilities**
- Normalize model and rules outputs to a 0–1 range.  
- Combine signals with stable weights to favor high precision early on.  
- Apply a **weekly-updated threshold** derived from recent normal behavior.  
- Emit final anomaly decision and confidence.

---

## Key Characteristics
- **Single-Client Simplicity**: No tenant routing or RLS; one deployment, one client.  
- **Hybrid Detection**: ML + rules for precision and adaptability.  
- **Retraining Flexibility**: Weekly schedule now; can add drift-triggered retraining later.  
- **Low Latency Goal**: Ingestion-to-alert under **2 seconds** (P95) on MVP hardware.  
- **Extensible**: Easy to add sources (batch, stream), models, and destinations (S3, CRM, BI).

---

## Next Steps
- Define **business KPIs** (alert precision, recall, latency SLA).  
- Set up CI/CD (build, test, deploy to staging).  
- Implement MVP with **global model + calibration layer** and a basic webhook receiver for testing.  
- Schedule the first **weekly retraining run** on synthetic + pilot data.  

*Tip for GitHub Mermaid:* use `<br/>` for line breaks inside node labels.

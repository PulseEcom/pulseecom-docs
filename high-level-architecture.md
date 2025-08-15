# PulseEcom – High-Level Architecture

## Overview
The PulseEcom platform is designed as a **real-time, multi-tenant analytics pipeline** for e-commerce businesses.  
It collects event data from client systems, processes and enriches it, applies AI/ML-based anomaly detection, and delivers actionable insights through APIs, AI summaries, and alerts.

---

## Architectural Goals
- **Real-Time Processing** – Minimal latency between event ingestion and insights.
- **Multi-Tenancy** – Secure data isolation for multiple clients.
- **Scalability** – Handle high event volumes from large e-commerce platforms.
- **Extensibility** – Add new data sources, detection models, and reporting features with minimal changes.
- **Integration-Friendly** – Support APIs, streaming, and batch ingestion.

---

## Key Components

### 1. API Gateway
- Single entry point for all API requests.
- Handles authentication (API keys), rate limiting, and request validation.
- Routes requests to appropriate services.

### 2. Ingestion Service
- Accepts event data from clients via REST API or batch upload.
- Validates and standardizes incoming data.
- Publishes raw events to **Kafka** (or Redpanda) for processing.

### 3. Enrichment Service
- Consumes raw events from Kafka.
- Applies **tenant-specific mapping rules** and adds derived fields (e.g., geolocation, device type).
- Publishes enriched events to the next Kafka topic.

### 4. Risk Scoring Service
- Calls the **ML Service** to detect anomalies or suspicious patterns.
- Annotates events with risk scores and anomaly flags.
- Persists scored events in **Postgres** for querying and reporting.

### 5. ML Service
- AI/ML models (e.g., IsolationForest) trained on historical client data.
- Detects deviations from normal behavior.
- Future models will include fraud detection, trend prediction, and campaign optimization.

### 6. AI Insights Service
- Fetches recent scored events and aggregates.
- Generates **plain-language summaries** highlighting key anomalies and trends.
- Uses tenant-specific prompt templates for clarity and relevance.

### 7. Query & Reporting Service
- Provides APIs to retrieve metrics, aggregates, and event history.
- Supports filtering by time range, campaign, store, and product.

### 8. Alerting Service
- Sends real-time notifications via webhooks, email, or messaging tools.
- Retries failed deliveries with exponential backoff.
- Supports replay of past alerts for compliance and analysis.

### 9. Postgres Database
- Stores all scored events, tenant configurations, usage data, and alert history.
- Implements **Row-Level Security (RLS)** for tenant isolation.

### 10. Monitoring & Observability
- Logs all service interactions with `tenant_id` and `trace_id`.
- Tracks key metrics like ingest rate, Kafka lag, anomaly detection accuracy, and webhook success rates.

---

## Data Flow Diagram

```mermaid
flowchart LR
    Client[Client Systems] -->|Events/API Calls| Gateway[API Gateway]
    Gateway --> Ingest[Ingestion Service]
    Ingest -->|Raw Events| Kafka[Kafka / Redpanda]
    Kafka --> Enrich[Enrichment Service]
    Enrich -->|Enriched Events| Kafka2[Kafka]
    Kafka2 --> Scorer[Risk Scoring Service]
    Scorer -->|Call ML Model| ML[ML Service]
    ML --> Scorer
    Scorer -->|Scored Events| DB[(Postgres)]
    Scorer -->|Scored Events| Kafka3[Kafka]
    Kafka3 --> Alerts[Alerting Service]
    DB --> Query[Query & Reporting Service]
    DB --> AIInsights[AI Insights Service]
    AIInsights --> Query
    Alerts --> ClientNotify[Client Alerts/Webhooks]
    Query --> ClientDash[Client Dashboards/Integrations]

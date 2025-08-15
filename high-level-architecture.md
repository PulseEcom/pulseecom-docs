# PulseEcom – High-Level Architecture

## Overview
The PulseEcom platform is a real time analytics pipeline for e commerce businesses.  
It collects event data from client systems, enriches it, detects anomalies, and delivers actionable insights via APIs, summaries, and alerts.

---

## Architectural Goals
- **Real time processing** with minimal latency from ingest to insight  
- **Single tenant MVP** now, expandable later  
- **Scalable and extensible** services and data flow  
- **Integration friendly** APIs, streaming, and batch support  

---

## Key Components
- **API Gateway** – Authentication, rate limiting, routing  
- **Ingestion Service** – Validates input and writes raw events to Kafka  
- **Enrichment Service** – Adds context like device, geo, category and republishes  
- **Scoring Layers** – Pre Trained Models, Client Model, Calibration Layer  
- **Query and Insights** – Metrics and summaries over scored data  
- **Alerting Service** – Webhook notifications with retries  
- **Postgres** – Storage for scored events and aggregates  
- **Learning and Model Update** – Retraining, drift checks, calibration updates  

---

## Data Flow Diagram

```mermaid
flowchart TD
    %% Layers
    subgraph Ingestion_Layer[Ingestion Layer]
        GW[API Gateway]
        IG[Ingestion Service]
    end

    subgraph Enrichment_Layer[Enrichment Layer]
        EN[Enrichment Service]
    end

    subgraph Scoring_Layer[Scoring Layer]
        PTM[Pre-Trained Model]
        CSM[Client-Specific Model]
        CAL[Calibration Layer]
    end

    subgraph Learning_Layer[Learning Layer]
        DRIFT[Drift Detection]
        RET[Retraining Job]
        THRES[Threshold Recalibration]
    end

    subgraph Persistence_Layer[Persistence Layer]
        DB[(Postgres Database)]
    end

    subgraph Delivery_Layer[Delivery Layer]
        AL[Alerting Service]
        QAPI[Query API]
        AIS[AI Insights Service]
    end

    %% Flow
    GW --> IG --> EN
    EN --> PTM
    EN --> CSM
    PTM --> CAL
    CSM --> CAL
    CAL --> DB
    CAL --> AL
    DB --> QAPI
    DB --> AIS
    AL --> ClientAlerts[Client Notifications]
    QAPI --> ClientDashboards[Client Dashboards]
    AIS --> ClientDashboards

    %% Learning feedback loop
    DB --> DRIFT --> RET
    RET --> CSM
    RET --> CAL
    THRES --> CAL



# PulseEcom – Low Level Design (MVP v0)

## Scope
Single client MVP covering Ingestion, Enrichment, Scoring, Alerting, Query, and Learning.  
One Postgres instance, one Kafka cluster.

---

## 1. Service Boundaries

### Ingestion Service
- Accepts events via REST.
- Validates payloads.
- Publishes to Kafka topic `events_raw`.

### Enrichment Service
- Consumes `events_raw`.
- Adds context fields.
- Publishes to `events_enriched`.

### Scoring Service
- Consumes `events_enriched`.
- Runs Pre-Trained Models and Client Model.
- Combines via Calibration Layer.
- Persists to Postgres table `events_scored`.
- Publishes to `events_scored` topic.

### Alerting Service
- Consumes `events_scored`.
- Filters anomalies.
- Sends signed webhook to client.

### Query API
- Read-only views on Postgres for metrics and details.

### Learning Job
- Batch job.
- Builds features, retrains Client Model, updates calibration thresholds.

---

## 2. External Contracts

### 2.1 Ingestion REST API

**Endpoint:**
```
POST /api/v1/events:batch
```

**Headers:**
```
Authorization: Bearer <API_KEY>
Idempotency-Key: <uuid>   # optional
```

**Body (max 1000 items):**
```json
{
  "events": [
    {
      "eventId": "e-001",
      "ts": "2025-08-15T10:20:05Z",
      "type": "order",
      "campaignId": "c-12",
      "storeId": "s-01",
      "amount": 149.90,
      "device": "web",
      "meta": {"sku": "SKU123", "currency":"USD"}
    }
  ]
}
```

**Responses:**
```
202 Accepted { "accepted": <n>, "rejected": <m> }
400, 401, 409 (duplicate idempotency key), 429, 500
```

### 2.2 Webhook to Client

**POST** `{CLIENT_WEBHOOK_URL}`

**Headers:**
```
X-Pulse-Signature: sha256=...   # HMAC over body with shared secret
Content-Type: application/json
```

**Body:**
```json
{
  "id": "alert-9f2a",
  "ts": "2025-08-15T10:20:30Z",
  "severity": "high",
  "eventType": "order",
  "score": 0.91,
  "anomaly": true,
  "reasons": ["amount_spike", "geo_mismatch"],
  "window": {
    "start": "2025-08-15T10:15:00Z",
    "end": "2025-08-15T10:20:00Z"
  },
  "sampleEventIds": ["e-001", "e-002"],
  "links": {
    "api": "/api/v1/events/alerts/alert-9f2a"
  }
}
```

---

## 3. Kafka Topics

- **events_raw**  
  key: random or campaignId  
  value: `EventV1` JSON

- **events_enriched**  
  value: `EventEnrichedV1`

- **events_scored**  
  value: `EventScoredV1`

**Retention:**  
raw – 24h, enriched – 24h, scored – 7d (DLQ topics mirror each)

---

## 4. Schemas

### 4.1 EventV1
```json
{
  "eventId": "string",
  "ts": "RFC3339",
  "type": "impression|click|order|return|custom",
  "campaignId": "string",
  "storeId": "string",
  "amount": "number",
  "device": "web|app|pos",
  "meta": { "additional": "fields" }
}
```

**Validation rules:**
- `eventId` required, unique per platform.
- `ts` within now minus 24h and now plus 5m.
- `amount` >= 0 when present.

### 4.2 EventEnrichedV1 (added fields)
```json
{
  "...": "EventV1",
  "geo": {"country": "IN", "region": "TN"},
  "deviceInfo": {"ua": "...", "bot": false},
  "derived": {
    "hour": 10,
    "dow": 5,
    "rolling": {"clicks5m": 120, "orders1h": 45}
  }
}
```

### 4.3 EventScoredV1
```json
{
  "eventId": "string",
  "ts": "RFC3339",
  "type": "string",
  "score_base": 0.0,
  "score_rules": 0.0,
  "score_client": 0.0,
  "score_final": 0.0,
  "anomaly": true,
  "reasons": ["text1", "text2"]
}
```

---

## 5. Database Model (Postgres)
```sql
CREATE TABLE events_scored(
  id BIGSERIAL PRIMARY KEY,
  event_id TEXT NOT NULL UNIQUE,
  ts TIMESTAMPTZ NOT NULL,
  type TEXT NOT NULL,
  payload JSONB NOT NULL,
  score_base DOUBLE PRECISION,
  score_rules DOUBLE PRECISION,
  score_client DOUBLE PRECISION,
  score_final DOUBLE PRECISION NOT NULL,
  anomaly BOOLEAN NOT NULL,
  reasons JSONB,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_events_scored_ts ON events_scored(ts);
CREATE INDEX idx_events_scored_anom_ts ON events_scored(anomaly, ts DESC);
```

**Materialized views for quick metrics:**
```sql
CREATE MATERIALIZED VIEW mv_metrics_5m AS
SELECT
  date_trunc('minute', ts) AS minute,
  type,
  COUNT(*) AS events,
  AVG(score_final) AS avg_score,
  SUM(CASE WHEN anomaly THEN 1 ELSE 0 END) AS anomalies
FROM events_scored
WHERE ts > now() - interval '7 days'
GROUP BY 1, 2;
```

---

## 6. Scoring Details

### 6.1 Models
- **Pre-Trained Models**
  - IsolationForest artifact `global_iforest_vN`
  - Seasonal baseline forecaster (STL/Prophet equivalent)
  - Robust Z-score with MAD for velocity checks
- **Client Model**
  - IsolationForest or Gradient Boosted Trees trained weekly
- **Calibration Layer**
  - Normalizes scores into 0–1
  - Weighted sum: `0.5 * base + 0.3 * rules + 0.2 * client`
  - Threshold recalibrated to target precision (default 0.80)

### 6.2 Threshold Recalibration
- Compute distribution of `score_final` for recent normal windows.
- Set threshold to p995 of normal or tuned to business precision target.
- Persist threshold with version: `calib_vN`.

---

## 7. Query API (read)

**Example:**
```
GET /api/v1/metrics/overview?from=...&to=...
```

**Response:**
```json
{
  "events": 145200,
  "anomalies": 230,
  "avgScore": 0.22,
  "byType": [
    {"type": "order", "events": 1200, "anomalies": 15}
  ]
}
```

Other endpoints:
```
GET /api/v1/events/scored?from=...&to=...&anomaly=true&limit=100&cursor=...
GET /api/v1/events/{eventId}
```

---

## 8. Alerting Logic
- **Filter:** anomaly is true AND `score_final` >= current threshold.
- Optional persistence rule: N of M windows.
- **Retry:** exponential backoff up to 5 attempts.
- Log to `webhook_deliveries` table.
- **HMAC signature:**  
  `signature = sha256_hex(secret, timestamp + "." + body)`

---

## 9. Security
- API key on ingestion and query.
- TLS everywhere.
- HMAC for outbound webhooks.
- Input schema validation and size limits.
- Rate limits per API key.

---

## 10. Observability
- **Logs (JSON):** service, trace_id, stage, latency_ms, status, error.
- **Metrics:** RED per service, Kafka consumer lag, alert delivery success rate, model versions in use.
- **Tracing:** propagate `trace_id` from Gateway to all services.

---

## 11. Sequences

```mermaid
sequenceDiagram
  participant C as Client
  participant G as Gateway
  participant I as Ingestion
  participant K as Kafka
  participant E as Enrichment
  participant S as Scoring
  participant D as Postgres
  participant A as Alerting

  C->>G: POST events batch
  G->>I: forward
  I->>K: produce events_raw
  K->>E: consume events_raw
  E->>K: produce events_enriched
  K->>S: consume events_enriched
  S->>D: insert events_scored
  S->>K: produce events_scored
  K->>A: consume events_scored
  A-->>C: webhook POST
```

---

## 12. Config Defaults
```ini
KAFKA_BOOTSTRAP=kafka:9092
TOPIC_RAW=events_raw
TOPIC_ENRICHED=events_enriched
TOPIC_SCORED=events_scored

DB_URL=postgres://app:app@postgres:5432/pulseecom
API_KEY=replace_me
WEBHOOK_URL=https://client.example.com/hooks/pulseecom
WEBHOOK_SECRET=replace_me
CALIB_TARGET_PRECISION=0.9
THRESHOLD_MIN=0.6
```

---

## 13. Non-Functionals
- Ingest to scored P95 under 900 ms.
- Alert delivery P95 under 150 ms after scoring.
- Availability 99.9% for ingest and alerting paths.
- Data retention 90 days (MVP).

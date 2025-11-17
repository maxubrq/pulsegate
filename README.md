# PulseGate

**Fast. Safe. Replayable.**
The minimal-but-ruthless event front-door.

---

## 🧩 Overview

PulseGate là **cổng ingest sự kiện** cho hệ thống hiện đại: an toàn, mở rộng được, và plugin-driven.
Nó nhận event từ upstream (HTTP/gRPC/Kafka), chuẩn hóa chúng thành **CanonicalEvent**, đưa vào ingestion log, áp dụng transform/plugins, và đẩy tới downstream sinks (Postgres, Kafka, Webhook, S3…).

**Ba nguyên tắc của PulseGate:**

1. **Không mất event**
2. **Không để một tenant phá cả hệ thống**
3. **Không bị ràng buộc bởi bất kỳ protocol hay storage cụ thể nào**

PulseGate nhỏ, nhưng từng chi tiết đều sắc.

---

## ✨ Features

### Core Reliability

* Durable ingestion log (Redis Streams hoặc Kafka)
* Idempotency (event_id + idempotency_key)
* Backpressure & flow control
* Multi-tenant QoS
* DLQ + replay theo time-range
* Stateless worker pipeline

### Extensibility

* **Plugin system** (Transform Plugins + Sink Plugins)
* Protocol-agnostic core (HTTP/gRPC/Kafka chỉ là adapters)
* Config per tenant: plugin chain, routing rules

### Observability

* Prometheus metrics
* Structured logs
* Queue lag
* Per-tenant throughput & error rate

### Downstream Agnostic

* Built-in sinks: Postgres, Kafka
* Easy to add more sinks (S3, Webhook, ClickHouse…)

---

## 🏛 Architecture

PulseGate có 3 lớp chính:

```
                    ┌─────────────────┐
     Upstream       │ Ingress Adapter │   (HTTP / gRPC / Kafka)
                    └───────┬─────────┘
                            ↓
                    ┌─────────────────┐
                    │ CanonicalEvent  │
                    └───────┬─────────┘
                            ↓
                    ┌─────────────────┐
                    │ Ingestion Log   │  (Redis Streams / Kafka)
                    └───────┬─────────┘
                            ↓
                    ┌──────────────────────────┐
                    │   Worker Pipeline        │
                    │   - QoS / Backpressure   │
                    │   - Transform Plugins    │
                    └───────┬─────────┬────────┘
                            │         │
                            ↓         ↓
                 ┌────────────────┐  ┌────────────────┐
                 │ Sink Plugin(s) │→│   DLQ / Replay  │
                 └────────────────┘  └────────────────┘
```

**Core chỉ làm việc với `CanonicalEvent`.**
Mọi protocol và mọi downstream đều qua adapter/plugin.

---

## 📦 Canonical Event

Định dạng bất biến:

```ts
type CanonicalEvent = {
  event_id: string;
  tenant_id: string;
  event_type: string;
  occurred_at: string;
  received_at: string;
  payload: Record<string, unknown>;
  source: "http" | "grpc" | "kafka";
  trace_id?: string;
  idempotency_key?: string;
};
```

---

## 🔌 Plugin System

### Transform Plugin

```ts
interface TransformPlugin {
  name: string;
  version: string;
  init?(config: unknown): Promise<void>;
  process(event: CanonicalEvent, ctx: PluginContext): Promise<
    | { type: "ok"; event: CanonicalEvent }
    | { type: "drop"; reason: string }
    | { type: "error"; reason: string; retryable: boolean }
  >;
}
```

Use cases:

* GeoIP enrichment
* PII scrubber
* Field normalization
* Business routing
* Bot detection

### Sink Plugin (Connector)

```ts
interface SinkPlugin {
  name: string;
  version: string;
  init?(config: unknown): Promise<void>;
  send(event: CanonicalEvent, ctx: PluginContext): Promise<
    | { type: "ok" }
    | { type: "retry"; reason: string }
    | { type: "error"; reason: string }
  >;
}
```

Built-in:

* `PostgresSink`
* `KafkaSink`

---

## ⚙️ Tenant Configuration

```yaml
tenants:
  - id: tenant_shop_1
    plugins:
      transform:
        - name: geoip_enricher
        - name: pii_scrubber
      sinks:
        - name: postgres_sink
          config:
            table: events_raw
        - name: kafka_sink
          config:
            topic: analytics_events
```

---

## 🚦 Reliability Guarantees

PulseGate đảm bảo:

* **At-least-once delivery**
* Durable commit trước khi ACK upstream
* Worker retry with backoff
* Transform failures không phá pipeline
* DLQ cho mọi event lỗi
* Replay event qua cùng pipeline và idempotent
* Per-tenant isolation (không “lây nhiễm” traffic)

---

## 📊 Metrics

PulseGate expose:

```
pulsegate_ingest_requests_total
pulsegate_ingest_latency_ms
pulsegate_stream_lag
pulsegate_worker_processing_time
pulsegate_worker_errors_total
pulsegate_dlq_events_total
pulsegate_tenant_backpressure_total
pulsegate_replay_events_total
```

Kèm log structured + trace_id.

---

## 🚀 Quick Start

**1) Start services**

```
docker compose up -d
```

Services:

* Redis Streams / Kafka
* Postgres
* PulseGate API
* Worker(s)

**2) Send event**

```bash
curl -X POST http://localhost:8080/v1/events \
  -H "X-API-Key: demo" \
  -H "Content-Type: application/json" \
  -d '{
    "event_type": "user_sign_up",
    "user_id": "u_123",
    "ip": "8.8.8.8",
    "timestamp": "2025-03-01T12:00:00Z"
  }'
```

**3) Query Postgres**

```sql
SELECT * FROM events_raw;
```

---

## 🧪 Testing Philosophy

* Worker and sink tested via integration tests
* Plugins tested with input → expected-event snapshot
* DLQ & replay tested via deterministic pipelines
* Load test (k6) 5k–10k events/s

---

## 📚 Project Structure

```
/pulsegate
  /adapters
    http/
    grpc/
    kafka/
  /core
    canonical/
    ingestion/
    router/
    qps/
  /plugins
    transform/
    sink/
  /workers
  /config
  /tests
```

Clean, predictable, dependency-light.

---

## 🧭 Roadmap

### v1 – Foundation

* Canonical event model
* HTTP ingest
* Ingestion log
* Worker pipeline
* Backpressure + QoS
* Postgres sink
* DLQ + replay
* Prometheus metrics

### v2 – Extensible

* Plugin system (transform + sink)
* Kafka sink
* Plugin registry
* Advanced routing
* Benchmarks + dashboards

### v3 – Enterprise

* gRPC ingress
* Kafka ingress
* S3 sink
* Schema evolution
* Admin API
* Replay UI

---

## 🧠 Philosophy

> **Make every detail perfect, but limit the number of details.**
> PulseGate không to, nhưng nó đúng.
> Không bóng bẩy, nhưng nó bền.
> Không “màu mè”, nhưng nó production-minded.

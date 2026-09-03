---
name: role-observability
description: Logs, metrics, and traces for production services: structured JSON logging, required log fields and levels, OpenTelemetry counters/histograms/spans, trace propagation, alert rules, and Kubernetes health checks. Use when instrumenting a service, debugging production, or setting up dashboards and alerting.
---

# Skill: Observability

## The three pillars

```
Logs    → WHAT happened (discrete events)
Metrics → HOW MUCH / HOW OFTEN (time series)
Traces  → WHERE it went (distributed flow)
```

Without all three, you're flying blind in production.

---

## Structured logging

Never free-text logs. Always structured JSON.

```python
import structlog
import logging

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ]
)

log = structlog.get_logger()

# ✅ Structured — searchable, filterable
log.info("user.login", user_id=user.id, ip=mask_ip(request.ip), success=True)
log.error("payment.failed", user_id=user.id, error_code="INSUFFICIENT_FUNDS", amount_cents=1000)

# ❌ Free text — impossible to analyze at volume
logging.info(f"User {user.email} logged in from {request.ip}")
```

### Mandatory fields in every log

```python
{
    "timestamp": "2026-04-20T14:32:00Z",  # ISO 8601
    "level": "info",
    "service": "user-service",             # service name
    "version": "1.4.2",                    # deployed version
    "request_id": "uuid",                  # per-request traceability
    "user_id": "abc",                      # business context (when applicable)
    "event": "user.login",                 # what happened (domain.action)
    "duration_ms": 45                      # latency when applicable
}
```

### Levels — when to use each one

| Level | When | Example |
|-------|--------|---------|
| `debug` | Dev only, never in prod | Internal state, intermediate variables |
| `info` | Normal business events | Login, payment processed, order created |
| `warn` | Something odd but not critical | Retry attempt 2/3, config fallback |
| `error` | Failure requiring attention | Exception, timeout, invalid data |
| `critical` | System compromised | DB down, secret rotated, service down |

---

## Metrics with OpenTelemetry

```python
from opentelemetry import metrics
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter

provider = MeterProvider()
metrics.set_meter_provider(provider)
meter = metrics.get_meter("user-service")

# Counters
requests_total = meter.create_counter(
    "http_requests_total",
    description="Total HTTP requests",
)

# Histograms (latency)
request_duration = meter.create_histogram(
    "http_request_duration_ms",
    description="HTTP request duration in milliseconds",
)

# Gauges (current state)
active_connections = meter.create_up_down_counter(
    "db_connections_active",
    description="Active database connections",
)

# Usage
def handle_request(method: str, path: str):
    start = time.time()
    try:
        result = process()
        requests_total.add(1, {"method": method, "path": path, "status": "200"})
        return result
    except Exception as e:
        requests_total.add(1, {"method": method, "path": path, "status": "500"})
        raise
    finally:
        duration = (time.time() - start) * 1000
        request_duration.record(duration, {"method": method, "path": path})
```

### Minimum metrics per service

```
http_requests_total{method, path, status}       # volume and error rate
http_request_duration_ms{method, path}          # latency (p50, p95, p99)
db_query_duration_ms{query_type}                # DB latency
db_connections_active                           # connection pool
cache_hits_total / cache_misses_total           # cache hit rate
queue_depth{queue_name}                         # queue depth
business_events_total{event_type}              # business metrics
```

---

## Distributed tracing with OpenTelemetry

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

provider = TracerProvider()
trace.set_tracer_provider(provider)
tracer = trace.get_tracer("user-service")

# Every significant operation = one span
def create_user(name: str, email: str) -> UserId:
    with tracer.start_as_current_span("create_user") as span:
        span.set_attribute("user.email_domain", email.split("@")[1])

        with tracer.start_as_current_span("db.save_user"):
            user_id = db.save(name, email)

        with tracer.start_as_current_span("email.send_welcome"):
            email_service.send_welcome(email)

        span.set_attribute("user.id", str(user_id))
        return user_id
```

**Trace propagation between services:**
```python
# Inject into outgoing HTTP headers
from opentelemetry.propagate import inject
headers = {}
inject(headers)
requests.post(url, headers=headers)

# Extract from incoming HTTP headers (middleware)
from opentelemetry.propagate import extract
context = extract(request.headers)
```

---

## Recommended stack

| Tool | What it does | When to use |
|-------------|----------|-------------|
| OpenTelemetry | Standard instrumentation | Always — it's the standard |
| Prometheus | Metrics scraping | Self-hosted / Kubernetes |
| Grafana | Dashboards | Self-hosted |
| Datadog | APM + logs + metrics | Managed, enterprise |
| AWS CloudWatch | Logs + metrics on AWS | If you're already on AWS |
| Jaeger / Tempo | Tracing | Self-hosted |

---

## Alerts — what to always monitor

```yaml
# Minimum alerting rules
- name: HighErrorRate
  condition: rate(http_requests_total{status="5xx"}[5m]) > 0.05
  message: "Error rate > 5% over the last 5 minutes"

- name: HighLatency
  condition: histogram_quantile(0.99, http_request_duration_ms) > 2000
  message: "p99 latency > 2s"

- name: DBConnectionPoolExhausted
  condition: db_connections_active / db_connections_max > 0.9
  message: "Connection pool at 90%"

- name: QueueDepthHigh
  condition: queue_depth > 10000
  message: "Queue backing up — consumer may be down"
```

---

## Health checks

```python
from fastapi import FastAPI
from enum import Enum

class HealthStatus(str, Enum):
    HEALTHY = "healthy"
    DEGRADED = "degraded"
    UNHEALTHY = "unhealthy"

@app.get("/health/live")   # Is the process alive? (Kubernetes liveness)
async def liveness():
    return {"status": HealthStatus.HEALTHY}

@app.get("/health/ready")  # Can it take traffic? (Kubernetes readiness)
async def readiness():
    checks = {
        "database": await check_db(),
        "cache": await check_redis(),
    }
    status = HealthStatus.HEALTHY if all(checks.values()) else HealthStatus.UNHEALTHY
    return {"status": status, "checks": checks}
```

---

## Common Observability decisions

Apply the decision protocol from CLAUDE.md when facing:
- **Stack:** OpenTelemetry + Grafana/Prometheus vs Datadog vs CloudWatch
- **Log aggregation:** ELK (Elasticsearch) vs Loki vs CloudWatch Logs
- **Alerting:** PagerDuty vs OpsGenie vs AlertManager
- **Trace sampling:** 100% vs probabilistic vs tail-based

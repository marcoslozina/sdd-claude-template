---
name: role-performance
description: Performance testing and profiling: load, stress, spike and soak tests with k6 or Locust, SLO-derived thresholds, cProfile bottleneck hunting, symptom-to-cause tables, and a performance stage in CI. Use when writing load tests, setting latency/throughput SLOs, or diagnosing slowness under load before a release.
---

# Skill: Performance Testing

## Core principle
Testing correctness is not testing performance. A system that passes every test
can still fall over with 100 concurrent users. Performance is tested explicitly.

---

## Types of performance tests

| Type | What it measures | When to run |
|------|----------|---------------|
| **Load test** | Behavior under expected load | Before every release |
| **Stress test** | The system's breaking point | When designing the architecture |
| **Spike test** | Response to sudden surges | For systems with variable traffic |
| **Soak test** | Degradation under sustained load (memory leaks) | Periodically in staging |
| **Baseline** | Reference metrics at rest | At the start of the project |

---

## k6 — main tool (recommended)

```javascript
// tests/performance/load_test.js
import http from 'k6/http'
import { check, sleep } from 'k6'
import { Rate, Trend } from 'k6/metrics'

// Custom metrics
const errorRate = new Rate('error_rate')
const createUserDuration = new Trend('create_user_duration')

export const options = {
  stages: [
    { duration: '2m', target: 10 },   // ramp up: 0 → 10 users
    { duration: '5m', target: 10 },   // sustained load: 10 users
    { duration: '2m', target: 50 },   // ramp up: 10 → 50 users
    { duration: '5m', target: 50 },   // sustained load: 50 users
    { duration: '2m', target: 0 },    // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% of requests < 500ms
    http_req_failed: ['rate<0.01'],    // < 1% errors
    error_rate: ['rate<0.05'],
  },
}

export default function () {
  // Create user
  const start = Date.now()
  const res = http.post(
    'http://localhost:8080/users',
    JSON.stringify({ name: 'Test User', email: `test_${__VU}_${__ITER}@test.com` }),
    { headers: { 'Content-Type': 'application/json' } }
  )

  createUserDuration.add(Date.now() - start)
  errorRate.add(res.status !== 201)

  check(res, {
    'status is 201': (r) => r.status === 201,
    'response has id': (r) => JSON.parse(r.body).id !== undefined,
    'duration < 500ms': (r) => r.timings.duration < 500,
  })

  sleep(1)
}
```

```bash
# Run the test
k6 run tests/performance/load_test.js

# With output to influxdb for Grafana
k6 run --out influxdb=http://localhost:8086/k6 tests/performance/load_test.js
```

---

## Locust — Python alternative

```python
# tests/performance/locustfile.py
from locust import HttpUser, task, between

class UserBehavior(HttpUser):
    wait_time = between(1, 3)  # wait between requests

    def on_start(self):
        # Setup per virtual user
        self.token = self.login()

    def login(self) -> str:
        res = self.client.post("/auth/login", json={
            "email": "test@test.com",
            "password": "testpass"
        })
        return res.json()["token"]

    @task(3)  # weight: runs 3x more often than other tasks
    def get_products(self):
        self.client.get("/products", headers={"Authorization": f"Bearer {self.token}"})

    @task(1)
    def create_order(self):
        self.client.post("/orders",
            json={"product_id": "abc-123", "quantity": 1},
            headers={"Authorization": f"Bearer {self.token}"}
        )
```

```bash
# Web UI at localhost:8089
locust -f tests/performance/locustfile.py --host=http://localhost:8080

# Headless
locust -f tests/performance/locustfile.py --host=http://localhost:8080 \
  --users 50 --spawn-rate 5 --run-time 10m --headless
```

---

## SLOs — define them before testing

```yaml
# Define the service SLOs before writing the tests
slos:
  availability: 99.9%          # at most 8.7h of downtime per year
  latency:
    p50: < 100ms
    p95: < 500ms
    p99: < 2000ms
  error_rate: < 0.1%
  throughput: > 1000 req/s     # minimum expected capacity
```

The k6/Locust thresholds must reflect these SLOs.

---

## Profiling — finding the bottleneck

```python
# Python: cProfile + snakeviz
import cProfile
import pstats

profiler = cProfile.Profile()
profiler.enable()

# code to profile
result = expensive_operation()

profiler.disable()
stats = pstats.Stats(profiler)
stats.sort_stats('cumulative')
stats.print_stats(20)  # top 20 slowest functions
```

```bash
# Visualize with snakeviz
pip install snakeviz
python -m cProfile -o output.prof my_script.py
snakeviz output.prof
```

### Signals of common problems

| Symptom | Likely cause | Investigate |
|---------|---------------|-----------|
| Latency grows over time | Memory leak | Heap profiler, GC metrics |
| High latency only at p99 | Outliers / GC pauses | Trace p99 requests |
| Low throughput with low CPU | I/O bound | Slow queries, external calls |
| Low throughput with high CPU | CPU bound | CPU profiler |
| Latency grows under load | Lock contention | DB connection pool, mutex |
| 503 errors under load | Capacity limit | Auto-scaling config |

---

## Performance in CI

```yaml
# .github/workflows/ci.yml — add a performance stage
performance:
  name: Performance Baseline
  runs-on: ubuntu-latest
  needs: integration-tests
  if: github.ref == 'refs/heads/main'
  steps:
    - uses: actions/checkout@v7
    - name: Start app
      run: docker compose up -d
    - name: Run k6 baseline
      uses: grafana/run-k6-action@v1
      with:
        path: tests/performance/baseline.js
    - name: Assert thresholds
      run: |
        # Fail the pipeline if the SLOs aren't met
        # k6 already handles this with the defined thresholds
        echo "Performance check complete"
```

---

## Checklist before release

- [ ] Load test at the expected load meets the SLOs
- [ ] Stress test identifies the breaking point (how much can it take?)
- [ ] No memory leaks in a 30-minute soak test
- [ ] Auto-scaling configured with the right metrics

DB indexing, query tuning, connection pooling and caching strategy are code-level
concerns — see `role-app-performance` for those.

---

## Common Performance decisions

Apply the decision protocol (this project's CLAUDE.md if it defines one, otherwise dev-harness's docs/DECISION_PROTOCOL.md) when facing:
- **Tool:** k6 vs Locust vs Gatling vs JMeter
- **Where to run:** local vs CI vs a dedicated environment
- **SLOs:** define p95 and p99 per operation type

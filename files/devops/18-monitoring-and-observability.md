# Monitoring & Observability

> **TL;DR:** Three pillars — metrics, logs, traces. Use Prometheus + Grafana for metrics, Loki/ELK/OpenSearch for logs, Jaeger/Tempo for traces, OpenTelemetry to instrument once and export everywhere. Frame your dashboards around USE (resources) and RED (services).

## Monitoring vs Observability

- **Monitoring:** are predefined symptoms occurring? (CPU > 80%, error rate > 1%). Reactive.
- **Observability:** can you ask **new questions** about your system after the fact, without redeploying? Cardinality, correlated signals, exploratory.

Monitoring is a subset; observability is the goal.

## The Three Pillars

| Pillar | What it answers | Tool examples |
|--------|-----------------|---------------|
| **Metrics** | "Is the rate / quantity within bounds?" | Prometheus, Datadog, CloudWatch |
| **Logs** | "What exactly happened to this request / event?" | Loki, ELK/EFK, OpenSearch, Splunk |
| **Traces** | "Where did time / errors go across services?" | Jaeger, Tempo, Zipkin, Datadog APM |

Bonus pillar people add: **Events** (deploys, incidents, scaling) — annotated on dashboards for correlation.

## USE and RED Methods

### USE — for **resources** (CPU, memory, disk, network)

For every resource:
- **Utilization** — % busy
- **Saturation** — queueing / waiting
- **Errors** — error count

### RED — for **services / requests**

For every service:
- **Rate** — requests per second
- **Errors** — failed requests per second (or %)
- **Duration** — request latency distribution

Combined with the **Four Golden Signals** (Google SRE book): latency, traffic, errors, saturation.

A good service dashboard has exactly RED on top, then drilldowns.

## Prometheus — the Pull-Based Standard

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: api
    static_configs:
      - targets: ['api:3000']
    metrics_path: /metrics

  - job_name: k8s-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Only scrape Pods with annotation prometheus.io/scrape=true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
```

Your app exposes `/metrics`:

```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",route="/api/users",status="200"} 12345
http_requests_total{method="GET",route="/api/users",status="500"} 7

# HELP http_request_duration_seconds Request duration
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.1"} 11000
http_request_duration_seconds_bucket{le="0.5"} 12200
http_request_duration_seconds_bucket{le="1"}   12340
http_request_duration_seconds_bucket{le="+Inf"} 12345
http_request_duration_seconds_sum 543.2
http_request_duration_seconds_count 12345
```

### Metric types

| Type | What | Example |
|------|------|---------|
| **Counter** | Monotonic, only goes up (or resets to 0) | `http_requests_total` |
| **Gauge** | Up and down | `memory_in_use_bytes`, `queue_depth` |
| **Histogram** | Bucketed distribution (compute quantiles in PromQL) | `request_duration_seconds` |
| **Summary** | Pre-computed quantiles on the client (don't aggregate well) | Avoid for distributed services |

### PromQL — the Queries You Actually Run

```promql
# Rate (per-second) of requests over 5 min
rate(http_requests_total[5m])

# Error rate %
sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
sum(rate(http_requests_total[5m])) * 100

# p99 latency from histogram (across instances)
histogram_quantile(0.99,
  sum by (le, route) (rate(http_request_duration_seconds_bucket[5m]))
)

# Increase over a window
increase(http_requests_total[1h])

# CPU usage % per pod (cAdvisor)
sum by (pod) (rate(container_cpu_usage_seconds_total[5m])) * 100

# Top 5 noisiest pods
topk(5, sum by (pod) (rate(container_cpu_usage_seconds_total[5m])))
```

**Cardinality is the killer.** Don't label by `user_id`, `request_id`, `trace_id` — each unique combination is a separate time series. 1M users × 10 metrics = 10M series → Prometheus melts.

## Alerting (Alertmanager)

```yaml
# rules.yaml
groups:
  - name: api-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{job="api",status=~"5.."}[5m]))
            /
          sum(rate(http_requests_total{job="api"}[5m])) > 0.02
        for: 10m
        labels:
          severity: page
        annotations:
          summary: "High 5xx rate on api"
          description: "5xx rate is {{ $value | humanizePercentage }} over 10m"
          runbook: "https://runbooks.example.com/api-high-5xx"
```

**Alert hygiene:**
- Alert on **symptoms (RED)**, not causes (CPU high). Cause goes in the runbook.
- Every page must be actionable. Otherwise it's a notification, not a page.
- `for:` prevents flapping.
- Include `runbook` link in annotations.
- Three severities max: **page** (wake me up), **ticket** (work hours), **info** (dashboard only).

Alertmanager handles routing to PagerDuty / Opsgenie / Slack / email, grouping, silences, inhibition (suppress alert B when A fires).

## Grafana — Dashboards

- Connect to Prometheus (and Loki, Tempo) as data sources.
- Build a service dashboard per service (RED + a few business KPIs at top).
- Don't make every team build their own from scratch — provide a templated "service dashboard" (variables: namespace, service).
- Annotate deploys and incidents.

Example panel queries:

```promql
# Traffic
sum by (route) (rate(http_requests_total{service="$service"}[1m]))

# Errors %
sum(rate(http_requests_total{service="$service",status=~"5.."}[5m]))
  /
sum(rate(http_requests_total{service="$service"}[5m]))

# Latency p50 / p95 / p99
histogram_quantile(0.50, sum by (le) (rate(http_request_duration_seconds_bucket{service="$service"}[5m])))
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket{service="$service"}[5m])))
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{service="$service"}[5m])))
```

## OpenTelemetry — One SDK to Rule Them All

OpenTelemetry (OTel) provides vendor-neutral SDKs + a Collector. Instrument once, export to anywhere (Prometheus, Jaeger, Tempo, Datadog, Honeycomb, ...).

```typescript
// Node.js — auto-instrumentation
import { NodeSDK } from "@opentelemetry/sdk-node";
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: "http://otel-collector:4318/v1/traces",
  }),
  instrumentations: [getNodeAutoInstrumentations()],
  serviceName: "api",
});
sdk.start();
```

The OTel Collector receives, batches, processes (sampling, attribute scrubbing), exports.

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols: { http: {}, grpc: {} }
processors:
  batch:
  tail_sampling:
    policies:
      - { name: errors, type: status_code, status_code: { status_codes: [ERROR] } }
      - { name: rate,   type: probabilistic, probabilistic: { sampling_percentage: 5 } }
exporters:
  prometheus:
    endpoint: 0.0.0.0:9464
  otlphttp/tempo:
    endpoint: http://tempo:4318
service:
  pipelines:
    traces:  { receivers: [otlp], processors: [tail_sampling, batch], exporters: [otlphttp/tempo] }
    metrics: { receivers: [otlp], processors: [batch], exporters: [prometheus] }
```

## Distributed Tracing in Practice

A **trace** = a tree of **spans** correlated by `trace_id`. Each span has start/end time, attributes, events, status.

```
TRACE  abcd1234
├── span: GET /api/orders      [api]         (250ms)
│   ├── span: SELECT orders    [postgres]    (40ms)
│   ├── span: GET /v1/user     [user-svc]    (80ms)
│   │   └── span: cache miss    [redis]       (5ms)
│   └── span: render template  [api]         (10ms)
```

Sampling: head-based (sample at root, simple) vs tail-based (decide after seeing the whole trace — keep errors, slow ones). Tail-sampling needs the Collector to buffer traces.

Propagate trace context across service calls via W3C `traceparent` header (auto-injected by OTel instrumentations).

## Logging vs Metrics vs Tracing — When Each

| Need | Use |
|------|-----|
| "How many 5xx in last hour?" | Metrics |
| "Why did THIS specific request fail?" | Logs (filtered by `trace_id`) |
| "Where did time go across these 8 services?" | Traces |
| "Show me all errors from user X" | Logs (with high-cardinality fields) |
| "Latency p99 trend by region" | Metrics |

## SLI / SLO / SLA — the Reliability Math

- **SLI** (Indicator): the *metric* (e.g., `successful_requests / total_requests`).
- **SLO** (Objective): your *target* (e.g., 99.9% over 30 days).
- **SLA** (Agreement): your *contract* with users (e.g., 99.5%, with penalty).

**Error budget** = 1 - SLO. With 99.9% SLO, you get 0.1% downtime / month ≈ 43 min. Use it strategically — gate risky deploys when burn rate is high.

```promql
# Error budget burn rate (multi-window, multi-burn-rate)
# Burn > 14.4 for 1h = 5%-budget-in-1h alarm
(
  sum(rate(http_requests_total{status=~"5.."}[1h]))
  /
  sum(rate(http_requests_total[1h]))
) / (1 - 0.999) > 14.4
```

## Cardinality Discipline

Bad metric labels (each unique combo = new time series):
- `user_id` (millions)
- `request_id` (unbounded)
- `email`
- `customer_uuid`

Good metric labels (low cardinality):
- `status_class` (`2xx`, `3xx`, `4xx`, `5xx`)
- `route` (templated, not raw URL)
- `region`
- `instance`

If you need per-request detail → that's logs/traces, not metrics.

## Interview Questions

**Q: USE vs RED?**
A: USE = Utilization/Saturation/Errors for resources (CPUs, disks). RED = Rate/Errors/Duration for services. Use RED on service dashboards, USE on host/node dashboards. Both are subsets of Google's Four Golden Signals (latency, traffic, errors, saturation).

**Q: Why does Prometheus pull instead of push?**
A: Pull simplifies service discovery (Prometheus knows who to scrape), avoids needing each instance to know push targets, and acts as a self-check (target unreachable → instant signal). Push is needed for short-lived batch jobs → use the Pushgateway.

**Q: SLI vs SLO vs SLA?**
A: SLI = the metric. SLO = your internal target. SLA = the external commitment with penalty if breached. Example: SLI is "p99 latency"; SLO is "p99 < 200ms 99.5% of the month"; SLA promises "99% availability" with refunds if breached.

**Q: How do you keep Prometheus from melting on cardinality?**
A: Don't label by high-cardinality fields (user_id, request_id). Use templated routes (`/users/:id` not `/users/12345`). Use recording rules to pre-aggregate hot queries. Move to long-term storage (Mimir, Thanos, Cortex) for horizontal scaling.

**Q: What's tail-based sampling and why use it?**
A: Decide to keep or drop a trace **after** seeing all its spans — so you can keep all error traces + slow ones, randomly sample the rest. Done in the OTel Collector. Head sampling (random at the root) is cheaper but loses interesting traces.

**Q: A service has high p99 latency intermittently. How do you investigate?**
A: (1) Confirm in metrics (RED dashboard). (2) Pull a slow trace from the relevant time window — find which span dominates. (3) If it's a downstream call, check that service's RED. (4) If it's the service itself, check logs around the slow trace ID. (5) Correlate with deploy events / saturation metrics.

## Common Pitfalls

- Alerting on causes ("CPU > 80%") instead of symptoms ("error rate > 1%") — pages you for non-problems.
- Using `user_id` as a Prometheus label — cardinality explosion.
- Average latency only — averages lie. Use p50/p95/p99.
- No deploy annotations on dashboards — can't correlate regressions with releases.
- Page fatigue — when every alert pages, on-call ignores them. Tune ruthlessly.
- Trace sampling at 100% — storage cost. Sample, but keep errors.
- Logging the world without structure (free text) — can't query, can't correlate.

## Related

- [19-logging-best-practices.md](19-logging-best-practices.md)
- [20-incident-response.md](20-incident-response.md)
- [06-kubernetes-fundamentals.md](06-kubernetes-fundamentals.md)
- [25-service-mesh-istio.md](25-service-mesh-istio.md)
- [../system-design/](../system-design/)

# Logging Best Practices

> **TL;DR:** Logs are structured JSON, written to stdout, shipped by a sidecar/daemon, indexed in a central store, correlated by `trace_id`. Levels are tuned (INFO baseline), retention is tiered (hot/warm/cold), and PII is scrubbed.

## The Rule of Thumb

```
container -> stdout/stderr -> kubelet / Docker -> log shipper -> central store -> dashboards + alerts
```

Don't log to files inside the container. Don't log to remote endpoints from within the app. Just print to stdout — the platform does the rest.

## Structured Logging

```json
{
  "ts": "2026-05-20T18:23:42.123Z",
  "level": "warn",
  "service": "api",
  "version": "1.4.0",
  "env": "prod",
  "trace_id": "abcd1234efgh5678",
  "span_id": "1234abcd",
  "user_id": 12345,
  "request_id": "req-7d8a9b",
  "route": "/api/orders/:id",
  "method": "POST",
  "status": 500,
  "duration_ms": 412,
  "msg": "order creation failed: payment gateway timeout",
  "error": {
    "type": "PaymentTimeoutError",
    "stack": "..."
  }
}
```

Why JSON?
- Indexable: filter by `level=error AND service=api`.
- Correlatable: pivot from a metric/trace to logs via `trace_id`.
- Type-safe: `duration_ms` stays a number; don't grep through text.

### Don't

```js
console.log(`User ${user.email} (${user.id}) did ${action} at ${new Date()}`);
```

### Do

```js
log.info({ user_id: user.id, action }, "user action");
```

PII like email goes into a dedicated audit log (with controls), not your general application log.

## Log Levels — Use Them Properly

| Level | When | Volume in prod |
|-------|------|----------------|
| `trace` | Wire-level detail | Off in prod |
| `debug` | Step-by-step dev info | Off in prod (toggle on for debugging) |
| `info` | Notable events: startup, request done, job started | High |
| `warn` | Recovered errors, fallbacks, deprecation | Low |
| `error` | Failed operation needing attention | Rare (every error is a candidate alert) |
| `fatal` | About to crash | Vanishingly rare |

If you log every `info` per request and have 10k RPS, that's 864M lines a day per service. Sample or downgrade.

**Dynamic log level** — let ops bump `LOG_LEVEL=debug` via env var or admin endpoint without redeploy. Critical during incidents.

## Correlation IDs

Every request should carry a stable `request_id` from edge → all downstream calls. Add to logs, traces, and propagate in headers.

```js
// Express middleware
import { randomUUID } from "crypto";

app.use((req, res, next) => {
  req.id = req.header("x-request-id") || randomUUID();
  res.setHeader("x-request-id", req.id);
  req.log = logger.child({ request_id: req.id, route: req.route?.path });
  next();
});

// Downstream call
await axios.get(url, { headers: { "x-request-id": req.id } });
```

If you also use OpenTelemetry, the `trace_id` serves as the correlation key — even better, since you can pivot to the trace too.

## What to Log (and What Not)

### Log

- Service start / config snapshot (no secrets).
- Each incoming HTTP request **at completion** (status, duration, route, request_id, user_id).
- Background job start/end + outcome.
- External API calls (URL, status, latency).
- Caught errors with stack trace.
- State transitions worth auditing (login, role change, payment).

### Don't Log

- Passwords, full credit cards, API keys, JWTs.
- Massive payloads (gigabytes/day).
- PII without a clear purpose (and matching access controls).
- Health check / liveness probe traffic (use sampling or filter).

### Audit Logs vs Application Logs

Separate stream / index / longer retention / stricter access. Audit logs answer "who did what when" for compliance (SOC2, HIPAA, GDPR).

## Shipping

### Kubernetes

Pods write to stdout/stderr. The kubelet captures it under `/var/log/pods/...`. A node-level DaemonSet ships to your backend:

- **Fluent Bit** (CNCF, lightweight, modern default).
- **Fluentd** (Ruby-based, very flexible, heavier).
- **Vector** (Rust, fast).
- **Promtail** (Loki's shipper).

```yaml
# Fluent Bit minimal config (excerpt)
[INPUT]
    Name              tail
    Path              /var/log/containers/*.log
    Parser            cri
    Tag               kube.*
    Refresh_Interval  5

[FILTER]
    Name              kubernetes
    Match             kube.*
    Merge_Log         On
    Keep_Log          Off
    K8S-Logging.Parser  On

[FILTER]
    Name              modify
    Match             *
    Remove            kubernetes.pod_id
    Remove            kubernetes.namespace_id

[OUTPUT]
    Name              loki
    Match             *
    Host              loki.observability.svc
    Port              3100
    Labels            job=fluentbit
    Auto_Kubernetes_Labels  on
```

## Storage Backends

| Backend | Strength | Watch out for |
|---------|----------|---------------|
| **Loki** | Label-based, S3 storage cheap, log-only "Prometheus-like" | Less powerful queries than full-text engines |
| **Elasticsearch / OpenSearch** | Powerful full-text, aggregations | Operational cost, schema explosions |
| **Datadog / Splunk / Logz.io** | Managed, all features | Per-GB cost; tune sampling |
| **CloudWatch Logs / Cloud Logging** | Cheap, native, decent search | Slow queries, harder to dashboard |

## Retention Tiering

```
last 7 days   -> hot index, full search
8-30 days     -> warm, slower search
31-90 days    -> cold (S3 / object storage), restore-on-demand
>90 days      -> compressed archive in cheapest storage, compliance only
```

Different log streams may have different retention. Application logs maybe 30d; audit logs 1-7 years.

## Cost Control

Logging is often the biggest observability bill. Levers:
- **Sampling:** keep all errors, sample info logs at 1-10%. Honor trace sampling — match log retention to trace decisions.
- **Drop chatty endpoints:** healthchecks, metrics scrapes.
- **Drop redundant fields:** the K8s log shipper may add 30+ labels. Keep what you query.
- **Compress** in shipping pipeline.
- **Right-size retention** per stream.

```promql
# Drop noisy in fluent bit (rewrite_tag) or Vector route — exclude /healthz, /readyz
```

## Query Examples

### Loki LogQL

```logql
{service="api", env="prod"} |= "payment" | json | level="error"

# Top errors by count
sum by (error.type) (count_over_time(
  {service="api"} | json | level="error" [5m]
))
```

### Elasticsearch / Kibana KQL

```
service: "api" and level: "error" and status: 500 and duration_ms > 1000
```

### Pivot from Trace to Logs

In a trace UI, click "view logs" → query by `trace_id` → see all the log lines from any service that handled that request. The killer observability move.

## Multi-Tenant / Multi-Team

- Tag every log line with `team` / `service` / `env` for routing and dashboards.
- Per-team retention + quotas to prevent one team's spam from sinking the bill for all.
- Index lifecycle policies (Elasticsearch ILM) to roll over and shrink.

## Interview Questions

**Q: Why structured JSON logs instead of free text?**
A: Queryability — filter by `level=error AND service=api AND status=500` in milliseconds. Correlation — pivot between metrics/traces/logs by `trace_id`. Types preserved (numbers stay numbers). Easier for machines, still readable by humans.

**Q: How do you correlate logs across microservices?**
A: Propagate a request/trace ID through every hop (`x-request-id` or W3C `traceparent`). Every log line includes it. Query by ID to assemble the full picture across services.

**Q: How do you control log volume in prod?**
A: Sample info-level logs (keep errors). Drop healthcheck noise. Use appropriate levels — info is the baseline, debug off. Quota by team. Tiered retention. Match log retention to trace sampling decisions.

**Q: Should you log every request?**
A: Yes — one access log line per request (at completion, with status + duration + route). It's the foundation of RED metrics if you ever lose metric ingestion. Use sampling for ultra-high RPS services.

**Q: How do you handle PII in logs?**
A: Don't log it. If you must, scrub at the source (mask email, tokenize). Use a separate, restricted audit log for user actions. Have a documented policy and access controls. GDPR/CCPA expect you can delete a user's data — easier if you didn't log it.

**Q: Loki vs Elasticsearch?**
A: Loki indexes labels only (cheap, "logs-as-streams"), backs to S3 — great for K8s logs at scale, integrates with Grafana. ES indexes content (more powerful queries, aggregations) — pricier, more ops. Use Loki for cheap volume; ES when you need full-text analytics.

## Common Pitfalls

- Logging passwords / tokens / full credit cards — appear in incident logs, compliance failure.
- Free-text logs — can't query, can't aggregate, can't correlate.
- No correlation ID — every incident is a manual jigsaw puzzle.
- Logging at debug in prod — drowns signal, costs $$$.
- Buffered stdout in Python — process crashes, last 50 lines never made it to disk. Use `PYTHONUNBUFFERED=1`.
- One giant log line of a serialized object — fills your index with junk you can't query.

## Related

- [18-monitoring-and-observability.md](18-monitoring-and-observability.md)
- [20-incident-response.md](20-incident-response.md)
- [21-security-devsecops.md](21-security-devsecops.md)
- [06-kubernetes-fundamentals.md](06-kubernetes-fundamentals.md)

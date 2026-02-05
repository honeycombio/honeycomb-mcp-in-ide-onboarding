# Honeycomb Concepts Reference

A quick-reference glossary of Honeycomb terminology and concepts. For learning explanations, see the onboarding guide.

---

## Core Data Concepts

### Event
The fundamental unit of data in Honeycomb. An event is a structured record of something that happened—typically a single operation or transaction. Each event has a timestamp and arbitrary key-value fields (columns).

### Span
A span is an event that represents one unit of work in a distributed trace. Spans have additional fields (`trace.trace_id`, `trace.span_id`, `trace.parent_id`) that link them into traces.

### Trace
A collection of spans that together represent a single request's journey through a distributed system. Visualized as a waterfall showing parent-child relationships and timing.

### Root Span
The first span in a trace—the one that initiated the request. Root spans have no `trace.parent_id`.

### Dataset
A collection of events sharing a schema. Typically maps to one service or application. Events within a dataset can be queried together.

### Environment
A logical grouping of datasets, usually corresponding to deployment stages (production, staging, dev). Teams can have multiple environments.

### Column / Field / Attribute
A key-value property on an event. Examples: `duration_ms`, `http.status_code`, `user.id`. Columns are the dimensions you filter, group by, and aggregate.

---

## Query Concepts

### VISUALIZE
The calculation to perform on matching events. Common operations:
- `COUNT` — Number of events
- `AVG(field)` — Average of a numeric field
- `P50(field)`, `P95(field)`, `P99(field)` — Percentiles
- `MAX(field)`, `MIN(field)` — Extremes
- `HEATMAP(field)` — Distribution visualization
- `SUM(field)` — Total of a numeric field

### WHERE
Filter clause that limits which events are included. Uses field comparisons:
- `service.name = "api-gateway"`
- `duration_ms > 1000`
- `http.status_code >= 500`

### GROUP BY
Splits results into buckets by field values. Shows breakdown like "count by endpoint" or "P95 by region."

### HAVING
Filters on aggregated values after grouping. Example: `HAVING COUNT > 100` shows only groups with more than 100 events.

### Granularity
The time bucket size for aggregations. Determines resolution of time-series charts.

### Heatmap
A visualization showing distribution of values over time. X-axis is time, Y-axis is value (e.g., latency), color intensity indicates count. Reveals patterns that averages hide.

---

## Percentiles

| Percentile | Meaning |
|------------|---------|
| P50 (median) | 50% of values are below this |
| P90 | 90% of values are below this |
| P95 | 95% of values are below this |
| P99 | 99% of values are below this |

**Why percentiles matter:** Averages hide outliers. P99 shows what your worst 1% of users experience.

---

## Tracing Concepts

### Trace ID
Unique identifier linking all spans in a trace. Format varies (usually UUID or hex string).

### Span ID
Unique identifier for a single span within a trace.

### Parent ID
The span ID of the span that called this one. Root spans have no parent.

### Duration
How long a span took to execute, typically in milliseconds (`duration_ms`).

### Service Name
The service that generated a span. Usually set via `service.name` or similar field.

### Span Name
The operation name for a span. Examples: `POST /checkout`, `db.query`, `redis.get`.

### Span Event
A timestamped annotation within a span. Often used for logging exceptions or significant events during the span's lifetime.

### Context Propagation
The mechanism for passing trace context (trace ID, span ID) between services, typically via HTTP headers or message metadata.

---

## SLO Concepts

### SLI (Service Level Indicator)
The metric being measured. Defined by a filter (which events) and a success criterion (what counts as good).

### SLO (Service Level Objective)
A target for the SLI over a time window. Example: "99.9% of checkout requests succeed over 30 days."

### SLA (Service Level Agreement)
A contractual commitment, typically to customers. SLOs should be tighter than SLAs to provide early warning.

### Error Budget
The amount of failure allowed while still meeting the SLO.
```
Error budget = (1 - target) × total events
```

### Budget Remaining
Percentage of error budget not yet consumed in the current window.

### Burn Rate
How fast the error budget is being consumed relative to the window duration.
- 1x = sustainable pace
- 10x = will exhaust budget 10x faster than sustainable

### Burn Alert
Notification triggered when burn rate exceeds a threshold, indicating SLO is at risk.

### Compliance
Current percentage of events meeting the SLI criteria.

---

## Analysis Concepts

### BubbleUp
Honeycomb's correlation analysis tool. Compares a selection of events (anomalies) against a baseline (everything else) to find which field values are over- or under-represented.

### Baseline
The reference point for comparison. Typically "all other events" for BubbleUp, or "same time yesterday" for temporal comparisons.

### Selection
The subset of events being analyzed (e.g., slow requests, errors).

### Outlier
An event or group of events that differs significantly from the norm.

### Core Analysis Loop
1. Observe anomaly in visualization
2. Select anomalous data
3. BubbleUp to find correlations
4. Drill into traces
5. Iterate with refined queries

---

## Instrumentation Concepts

### OpenTelemetry (OTel)
Open-source standard for telemetry data collection. Honeycomb recommends OTel for instrumentation.

### Auto-Instrumentation
Automatically generated spans from instrumentation libraries, covering common operations (HTTP requests, database queries, etc.).

### Manual/Custom Instrumentation
Developer-added spans and attributes for application-specific context.

### OTLP (OpenTelemetry Protocol)
The wire protocol for sending telemetry data. Honeycomb accepts OTLP over gRPC and HTTP.

---

## Field Naming Conventions

Honeycomb (via OpenTelemetry) uses namespaced field names:

| Namespace | Examples |
|-----------|----------|
| `trace.*` | `trace.trace_id`, `trace.span_id`, `trace.parent_id` |
| `service.*` | `service.name`, `service.version` |
| `http.*` | `http.method`, `http.status_code`, `http.route` |
| `db.*` | `db.system`, `db.statement`, `db.operation` |
| `error` | Boolean indicating if span represents an error |
| `name` | The span/operation name |
| `duration_ms` | Span duration in milliseconds |

---

## Quick Reference

### Essential Fields for Tracing
- `trace.trace_id` — Links spans into traces
- `trace.span_id` — Unique span identifier
- `trace.parent_id` — Parent span (null for root)
- `service.name` — Which service
- `name` — Operation name
- `duration_ms` — How long it took

### Common Query Patterns
```
# Count by status code
VISUALIZE: COUNT
GROUP BY: http.status_code

# P95 latency by endpoint
VISUALIZE: P95(duration_ms)
GROUP BY: http.route

# Error rate over time
VISUALIZE: COUNT
WHERE: http.status_code >= 500

# Latency distribution
VISUALIZE: HEATMAP(duration_ms)
```

### Time Shortcuts
| Shorthand | Meaning |
|-----------|---------|
| Last 1h | Past hour |
| Last 24h | Past day |
| Last 7d | Past week |
| Last 30d | Past month |

---

## Further Reading

- [Honeycomb Documentation](https://docs.honeycomb.io)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Honeycomb Learn Resources](https://www.honeycomb.io/resources)

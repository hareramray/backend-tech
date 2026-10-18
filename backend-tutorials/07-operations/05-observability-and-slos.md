# Observability, service objectives, and actionable alerts

[Tutorial index](../README.md) · Related: [incident response](07-incident-response-and-production-runbooks.md)

## Outcome and prerequisites

Define what healthy service means to a user, then collect evidence that explains deviations. You need Django request handling, HTTP status codes, and basic statistics. The logging example requires only the capstone and Python's standard library. Prometheus, Grafana, and OpenTelemetry examples describe optional extensions that require their own installed infrastructure and application instrumentation.

Observability is the ability to infer internal behavior from emitted evidence. Metrics summarize populations, logs describe selected events, and traces connect work across boundaries. Profiles describe resource use. These signals complement one another: a metric can identify a latency increase, a trace can locate a slow SQL operation, and a query plan can explain it.

## Start with user-visible outcomes

An SLI is a measured indicator, such as the fraction of eligible task-read requests that succeed. An SLO is its target over a window, such as 99.9% successful eligible requests over 30 days. An SLA is a contractual commitment with defined consequences; it is not interchangeable with an internal objective.

Specify numerator, denominator, measurement point, exclusions, window, and missing-data behavior. For task writes, an HTTP response alone may not prove durable success. Consider a separate correctness indicator or reconciliation process. Exclude intentional client validation failures only if the objective definition says so; do not hide server-caused failures by reclassifying them as client errors.

At a 99.9% success objective, the error budget is 0.1% of eligible requests. For one million requests, that permits 1,000 bad outcomes within the defined window. This request budget is not automatically equivalent to a fixed number of downtime minutes because traffic varies.

## Walkthrough: add minimal structured request logs

1. In a learning copy of the capstone, create `tasks/observability.py` with this synchronous Django middleware. It generates its own correlation ID and logs metadata rather than request content.

```python
import json
import logging
import time
import uuid

logger = logging.getLogger("task_service.requests")


class RequestLogMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        started = time.perf_counter()
        request_id = str(uuid.uuid4())
        request.request_id = request_id
        status = 500
        try:
            response = self.get_response(request)
            status = response.status_code
            response["X-Request-ID"] = request_id
            return response
        finally:
            match = request.resolver_match
            route = match.route if match is not None else "unmatched"
            logger.info(json.dumps({
                "event": "http_response_prepared",
                "request_id": request_id,
                "method": request.method,
                "route": route,
                "status": status,
                "duration_ms": round((time.perf_counter() - started) * 1000, 2),
            }))
```

2. Add `tasks.observability.RequestLogMiddleware` to `MIDDLEWARE` in the learning settings and configure a console handler for this named logger at `INFO` with `propagate=False`. In the capstone's `api` decorator, replace its independent `request_id = str(uuid.uuid4())` assignment with `request_id = request.request_id`. This makes error bodies, response headers, and both logging paths use the middleware's single ID. Handler configuration is necessary; merely creating a logger does not guarantee its events will be emitted. Use this middleware or the Django middleware chapter's alternative, not both.

3. Request `/v1/tasks/`, then an unknown route. Expect correlation headers and log records with the corresponding statuses. Verify that titles, passwords, cookies, authorization headers, and query strings do not appear. This middleware measures response preparation; streaming-body completion and client disconnects require server-level or streaming-aware instrumentation.

4. Forward logs through a controlled collector with retention, access control, and redaction. Add deployment revision and service identity at the collector or application configuration. Avoid embedding secrets in exception messages that later bypass field-based redaction.

## Metrics and cardinality

Instrument request count, duration histograms, active work, database connections, worker saturation, queue age, and external-call failures. Use stable labels such as service, route template, method, and bounded outcome. Never label metrics by task ID, user ID, raw URL, request ID, or arbitrary exception text. Every unique label combination creates another time series, multiplying storage and query cost. [Prometheus instrumentation guidance](https://prometheus.io/docs/practices/instrumentation/)

For a classic histogram with documented buckets, an example PromQL query is:

```promql
histogram_quantile(
  0.95,
  sum by (le) (rate(http_request_duration_seconds_bucket{job="task-service"}[5m]))
)
```

This assumes that exact metric and labels have been implemented and scraped; the capstone does not provide them automatically. Histogram bucket boundaries affect approximation quality. Aggregate bucket counts before computing a cross-instance percentile. Do not average per-instance p95 values.

Define eligible request counters consistently. For an illustrative good/bad counter, the bad-event ratio is:

```promql
sum(rate(task_requests_total{outcome="bad"}[5m]))
/
sum(rate(task_requests_total{outcome=~"good|bad"}[5m]))
```

Handle zero traffic and absent telemetry separately instead of interpreting missing data as perfect service. In a multi-service monitoring system, add bounded service or environment selectors so unrelated traffic cannot contaminate the result.

## Tracing and context propagation

A trace consists of spans representing work with parent-child relationships. Propagate context across HTTP calls and messages to connect causal operations. Record useful bounded attributes, SQL operation names, and error classifications; avoid secrets and raw personal data. OpenTelemetry defines trace and span concepts independently from a particular monitoring vendor. [OpenTelemetry traces](https://opentelemetry.io/docs/concepts/signals/traces/)

Sampling controls cost but can hide rare failures. Head sampling decides early; tail sampling can use completed trace information but requires buffering and appropriate collector design. Ensure traces from unsampled requests do not become a false denominator for an availability SLI. Metrics usually supply the population count.

## Alerting and verification

Burn rate is observed bad-event ratio divided by the allowed bad-event ratio. At a 99.9% objective, a 1% observed failure ratio burns budget at ten times the sustainable rate. Multiwindow alerts can require both a short and longer window to exceed thresholds, reducing noise while detecting sustained impact. Choose thresholds from the objective and response urgency, not arbitrary CPU percentages. [Google SRE alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)

Page when timely human intervention is needed. Route slower issues to tickets and informational events to dashboards. Every page needs an owner, user-impact description, evidence link, and runnable investigation path. A separate telemetry-absence alert protects against a broken monitoring pipeline.

In a lab, trigger controlled HTTP failures, stop a metrics exporter, and verify distinct alerts with correct routing. Exercise alert recovery and duplicate suppression. Common mistakes are logging secrets, treating health checks as user traffic, creating unbounded labels, and paging on every transient resource spike.

Finish by writing one availability SLO and one latency SLO for task reads, including exclusions and low-traffic behavior. Build a dashboard showing traffic, errors, latency, saturation, deployment events, and database health. Link each signal to a decision an operator can make.

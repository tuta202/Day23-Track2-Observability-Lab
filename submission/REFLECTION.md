# Day 23 Lab Reflection - Observability Stack

## 1. Tail-sampling math and logic
The system uses a tail-sampling policy in the OTel Collector with the following logic:
- Keep 100% of traces with errors (status_code == ERROR) or slow duration (duration > 2s).
- Keep a random sample of 1% for healthy traces.

Math: If the system processes 10,000 requests/sec with 0.1% errors (10 reqs/s).
- Without sampling: Storing 10,000 traces/s (huge storage and bandwidth cost).
- With this policy: Storing 10 error traces + (9,990 * 1%) approx 110 traces/s.
- Result: Reduces storage by approx 90x while maintaining full visibility into errors and a representative sample of normal traffic.

## 2. Structured Log Line with Trace Correlation
Here is a JSON log line from the day23-app container:

```json
{"model": "llama3-mock", "input_tokens": 4, "output_tokens": 54, "quality": 0.82, "duration_seconds": 0.1543, "trace_id": "4e0cbeed43b1688bedcabe98cc1e8363", "event": "prediction served", "level": "info", "timestamp": "2026-05-11T03:37:55.213045Z"}
```

Including the trace_id in the log allows Grafana to automatically create a "Jaeger" link next to the log line, enabling an engineer to jump directly from a log line to the full trace flame graph.

## 3. Drift Detection: PSI vs KL vs KS vs MMD
In this lab, we observed significant drift in prompt_length (PSI=3.461) and response_quality (PSI=8.849).

- PSI (Population Stability Index): Best for measuring overall distribution shifts in categorical or binned data. A value > 0.2 indicates a significant change.
- KL Divergence: Measures information loss when using one distribution to represent another. Sensitive to small changes but harder to interpret than PSI.
- KS Test (Kolmogorov-Smirnov): Best for continuous numerical data. It finds the maximum distance between two cumulative distribution functions (CDF). It does not require binning.
- MMD (Maximum Mean Discrepancy): A kernel-based method effective for multivariate data or complex structural changes that traditional statistical tests might miss.

## 4. Prior-day Integration: The Hardest Metric
The hardest metric to expose was from Day 20 (llama.cpp).
Reason: The llama.cpp server does not natively support Prometheus metrics. We had to use a sidecar script to parse logs or a stub monitor to simulate indicators like tokens_per_second. Mapping metrics from a black-box system to OpenTelemetry format requires a deep understanding of the system's behavior.

## 5. The single change that mattered most
The most important change was the Correlation between Logs and Traces.
Previously, when an error occurred, an engineer had to copy the timestamp from logs and manually search in Jaeger. Automatically attaching a trace_id to every log line and integrating a direct link in the UI (Grafana Loki to Jaeger) turned a "needle in a haystack" search into a single click. This reduces MTTR (Mean Time To Recovery) from minutes to seconds, allowing the team to focus on fixing the issue rather than finding it.

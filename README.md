# jmeter-test

Reference performance test repository for the **NextOps Performance Testing plugin**.

Trigger a JMeter load test against any HTTP target via the NextOps dashboard — no local JMeter install needed. Results are parsed and surfaced as pass/fail verdicts directly in the platform.

---

## Quick start

1. **Via NextOps UI** — navigate to *Perf Tests → New Test*, point it at this repo, pick a workflow and test plan, configure your target, and click *Save & Run*.
2. **Via GitHub Actions UI** — go to *Actions → Performance Test → Run workflow* and fill in the inputs manually.

---

## Repository structure

```
.github/
  workflows/
    perf-test.yml          ← workflow_dispatch workflow (required by NextOps)
scripts/
  collect-metrics.sh       ← generates nextops-metrics/v1 JSON from JTL results
tests/
  plans/
    sample-api-test.jmx    ← parameterised JMeter test plan (ready to use)
```

---

## Workflow inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `target_url` | yes | `https://httpbin.org` | Full URL of the target (e.g. `https://api.example.com`) |
| `test_plan` | yes | `tests/plans/sample-api-test.jmx` | Path to the JMX file in this repo |
| `users` | yes | `10` | Virtual user count |
| `ramp_up_seconds` | yes | `30` | Ramp-up period in seconds |
| `duration_seconds` | yes | `60` | Steady-state test duration in seconds |

---

## Artifacts produced

| Artifact | Contents | Required |
|---|---|---|
| `jmeter-results` | `results.jtl` (CSV) + `html-report/` directory | **Yes** — parsed by NextOps for pass/fail verdict |
| `metrics-results` | `metrics-results/metrics-results.json` (nextops-metrics/v1) | No — adds infrastructure charts to the report; plugin degrades gracefully without it |

---

## Adding your own test plans

1. Place your `.jmx` file under `tests/plans/`.
2. Parameterise the host, port, and protocol using JMeter properties:
   ```xml
   <stringProp name="HTTPSampler.domain">${__P(target_host,localhost)}</stringProp>
   <stringProp name="HTTPSampler.port">${__P(target_port,80)}</stringProp>
   <stringProp name="HTTPSampler.protocol">${__P(target_protocol,http)}</stringProp>
   ```
   The workflow parses `target_url` and passes `target_host`, `target_port`, and `target_protocol` automatically.
3. Use `${__P(users,10)}`, `${__P(rampup,30)}`, and `${__P(duration,60)}` in your Thread Group for the workload profile.

---

## Real infrastructure metrics (optional)

`scripts/collect-metrics.sh` currently derives synthetic CPU/memory/error-rate/latency series from the JTL file. To replace these with real Prometheus metrics:

1. Add your Prometheus URL as a repository secret named `PROMETHEUS_URL`.
2. In the script, set `PROMETHEUS_URL="${PROMETHEUS_URL:-}"` and uncomment the query block.
3. Use the `nextops-metrics/v1` schema — the plugin renders any series you include.

---

## nextops-metrics/v1 schema

```json
{
  "schema": "nextops-metrics/v1",
  "window": { "start": "ISO-8601", "end": "ISO-8601" },
  "series": {
    "my_metric": {
      "label": "Human-readable label",
      "unit": "% | ms | req/s | …",
      "threshold": { "type": "fail | alert", "max": 80 },
      "points": [{ "t": "ISO-8601", "v": 42.1 }]
    }
  }
}
```

- `threshold.type = "fail"` contributes to the overall PASS/FAIL verdict.
- `threshold.type = "alert"` is shown as a warning line on the chart but does not fail the run.

---

## JTL thresholds

Thresholds are configured in the NextOps UI when creating a test configuration:

| Threshold | Default | Description |
|---|---|---|
| Error rate | 1 % | Percentage of failed samples |
| P99 latency | 2000 ms | 99th-percentile response time |

---

## Presets (available in the wizard)

| Preset | Users | Ramp-up | Duration |
|---|---|---|---|
| Smoke | 5 | 10 s | 60 s |
| Load | 50 | 60 s | 300 s |
| Stress | 200 | 120 s | 600 s |

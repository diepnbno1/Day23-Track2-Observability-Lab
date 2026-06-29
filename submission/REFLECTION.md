# Day 23 Lab Reflection

**Student:** NGUYỄN BÁCH ĐIỆP - 2A202600535
**Submission date:** 2026-06-29
**Lab repo URL:** TODO - replace with your public GitHub fork URL

---

## 1. Hardware + setup output

Output of `python 00-setup/verify-docker.py`:

```json
{
  "docker": {"ok": true, "version": "29.5.3"},
  "compose_v2": {"ok": true, "version": "5.1.4"},
  "ram_gb_available": 7.69,
  "ram_ok": true,
  "required_ports": [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888],
  "bound_ports": [],
  "all_ports_free": true
}
```

Evidence file: `00-setup/setup-report.json`.

---

## 2. Track 02 - Dashboards & Alerts

Screenshots:

- `submission/screenshots/dashboard-overview.png`
- `submission/screenshots/cost-and-tokens.png`
- `submission/screenshots/slo-burn-rate.png`
- `submission/screenshots/alertmanager-firing.png`
- `submission/screenshots/alertmanager-resolved.png`

Load run: baseline Locust at 10 users for 60s, then an error-load run with `ERROR_RATE=0.2` to populate SLO burn-rate panels. The overview dashboard shows request rate, latency percentiles, error rate, GPU gauge, token throughput, and in-flight requests. The cost dashboard shows non-zero estimated spend; the SLO dashboard shows burn-rate lines after injected expected 503s.

Alert drill:

| When | What | Evidence |
|---|---|---|
| T0 | stopped `day23-app` | `alertmanager-firing.png` |
| T0+~120s | `ServiceDown` fired | Alertmanager UI showed `ServiceDown`; Alertmanager logs had no Slack send errors |
| T1 | restarted `day23-app` | `docker start day23-app` |
| T1+~60s | alert resolved | `alertmanager-resolved.png` |

One thing that surprised me: the `for: 1m` alert can take closer to two minutes end-to-end because the scrape interval, rule evaluation interval, and Alertmanager group wait all stack together. That is a good reminder that alert latency is not only the rule threshold; it is the whole path from scrape to notification.

---

## 3. Track 03 - Tracing & Logs

Trace screenshot:

- `submission/screenshots/jaeger-trace.png`

Trace retained by tail sampling:

- trace id: `b4ebe429882dd427142a08ecaf121771`
- parent span: `predict`, duration about `2.1s`
- child spans: `embed-text`, `vector-search`, `generate-tokens`

GenAI attributes observed through the Jaeger API:

- `predict`: `gen_ai.request.model=llama3-mock`
- `generate-tokens`: `gen_ai.usage.input_tokens=5`, `gen_ai.usage.output_tokens=10`, `gen_ai.response.finish_reason=stop`

Structured JSON log correlated to the trace:

```json
{"model":"llama3-mock","input_tokens":5,"output_tokens":10,"quality":0.796,"duration_seconds":2.0981,"trace_id":"b4ebe429882dd427142a08ecaf121771","event":"prediction served","level":"info","timestamp":"2026-06-29T11:35:28.751623Z"}
```

Tail-sampling math: during the main load, the service handled roughly 17 req/s. The policy keeps 1% of healthy traces, so healthy retained traffic is about `17 * 0.01 = 0.17 traces/s`, plus 100% of error traces and 100% of slow traces over 2s. The trace above used a deterministic slow prompt (`day23 slow trace candidate 2086`), so it was retained by the `keep-slow` policy instead of relying on random 1% sampling.

---

## 4. Track 04 - Drift Detection

Screenshots/files:

- `submission/screenshots/drift-evidently-report.png`
- `04-drift-detection/reports/drift-summary.json`
- `04-drift-detection/reports/drift-report.html`

`drift-summary.json`:

```json
{
  "prompt_length": {"psi": 3.461, "kl": 1.7982, "ks_stat": 0.702, "ks_pvalue": 0.0, "drift": "yes"},
  "embedding_norm": {"psi": 0.0187, "kl": 0.0324, "ks_stat": 0.052, "ks_pvalue": 0.133853, "drift": "no"},
  "response_length": {"psi": 0.0162, "kl": 0.0178, "ks_stat": 0.056, "ks_pvalue": 0.086899, "drift": "no"},
  "response_quality": {"psi": 8.8486, "kl": 13.5011, "ks_stat": 0.941, "ks_pvalue": 0.0, "drift": "yes"}
}
```

Which test fits which feature:

- `prompt_length`: PSI is good for monitoring distribution shift in binned numeric production features, and KS is a useful backup because this is continuous.
- `embedding_norm`: KS is the clearest check because it is a continuous scalar and small shifts may matter before buckets move much.
- `response_length`: PSI is useful for operational monitoring because response length can be bucketed into interpretable ranges; KS can validate continuous distribution movement.
- `response_quality`: KS and PSI both work, but I would alert on PSI/KS together because quality is bounded `[0,1]` and a shape change can matter even if the mean looks acceptable. For full embedding vectors, MMD would fit better than these one-dimensional tests.

---

## 5. Track 05 - Cross-Day Integration

Screenshot:

- `submission/screenshots/cross-day-dashboard.png`

I used stub sources for Day 19 and Day 20 because the optional Day19/Day20 URLs were intentionally left empty. Prometheus now scrapes `day19-stub:9101` and `day20-stub:9102`; the cross-day dashboard shows Day 19 Qdrant collection count and Day 20 llama.cpp token/sec data, while other prior-day panels fail soft with "No Data".

The hardest prior-day metric would be Day 20 serving metrics, because the Python llama-cpp server path does not expose `/metrics`; it requires the native `llama-server --metrics` path or a sidecar. That is exactly the kind of integration edge that observability makes visible.

---

## 6. The single change that mattered most

The single change that mattered most was making telemetry identifiers stable and connected: Grafana dashboards now use a stable Prometheus datasource UID, and the FastAPI trace code now creates `predict` as the active parent span before `embed-text`, `vector-search`, and `generate-tokens`. Before that, the system technically emitted data, but the dashboard panels could not resolve their datasource and Jaeger showed separate orphan spans. After the fix, the stack became operable: dashboards answered "what is happening?" and Jaeger answered "why did this request take 2.1s?"

This maps directly to the deck's observability theme: signals are only useful when they correlate. RED metrics showed the error/load shape, token and cost metrics exposed the AI-specific fourth pillar, and the trace/log shared `trace_id` gave a path from an aggregate symptom to one concrete request trajectory. That connection is the difference between "we collected telemetry" and "an on-call person can debug the system."

# Monitoring and Observability
# Splunk, Grafana, logs, metrics, traces, alerts

Use this to prove an incident with numbers, not only a single stack trace.
Three signals: logs (what happened), metrics (how much / how fast), traces (one request across services).

---

## 1. What to ask of each tool

| Question | Tool |
|---|---|
| Which error code started at 12:07, and how many? | Splunk or the log file |
| Did CPU, heap, or latency move at the same minute? | Grafana |
| Which service after the gateway returned 400? | Trace or correlation id across logs |
| Is the alert still firing? | Grafana alert or Splunk saved search |

A single ERROR line is a clue. A count by minute shows whether it is one customer or a platform fault.

---

## 2. Correlation id

Every hop should log the same id. Names you will see: `correlationId`, `X-Correlation-Id`, `traceId`, `Txid`.

When you only have a gateway line, copy the id and search again. The downstream microservice log is the one with the business reason.

If the id is missing, search by time window plus MSISDN or subscriber id. Keep the window tight (5–15 minutes) or the search will be slow and noisy.

---

## 3. Splunk searches

Index and sourcetype names differ by estate. Replace `index=app` and `sourcetype=gateway`.

Count errors by minute:

```spl
index=app sourcetype=gateway earliest=-2h
| search "ERROR" OR "CPG-"
| timechart span=1m count
```

Top inner error codes:

```spl
index=app earliest=-4h "CPG-"
| rex "code\"?:?\"?(?<code>CPG-[A-Z]+-\d+)"
| stats count by code
| sort -count
```

One MSISDN:

```spl
index=app earliest=-24h "998905904003"
| table _time host sourcetype correlationId status code
| sort _time
```

One correlation id:

```spl
index=app earliest=-4h "b7b66866-7e52-4582-ba73-77561722757d"
| table _time host source status code reason
| sort _time
```

Slow calls:

```spl
index=app sourcetype=access earliest=-1h
| where duration_ms > 3000
| stats count avg(duration_ms) max(duration_ms) by uri
| sort -avg(duration_ms)
```

Compare to yesterday, same hour:

```spl
index=app earliest=-1h ERROR
| timechart span=5m count
| timewrap 1d
```

Useful commands: `stats`, `timechart`, `rex`, `table`, `where`, `dedup`, `sort`.
A search that starts with a specific id or code is cheaper than `index=*`.

---

## 4. Grafana

Open the service dashboard for the same time range as the log.

Check together:

- request rate
- error rate (4xx vs 5xx separately)
- latency p95 / p99
- CPU and throttling
- JVM heap and GC time
- thread pool active vs max
- JDBC pool active vs max
- pod restarts
- Kafka consumer lag, if the app consumes

How to read a spike:

| Pattern | Likely cause |
|---|---|
| Error rate up, latency flat, CPU flat | Bad requests or a data defect |
| Latency and pool active both up | Downstream slow; threads waiting |
| CPU and GC time up, latency up | Heap pressure |
| Restarts up, then errors | Crash, OOM, or failed probe |
| Lag up, consumer count 0 | Consumer down |
| Lag up, consumer count normal | Consumer too slow or stuck on one partition |

Zoom to the minute the alert fired. Then look 15 minutes before that for the first movement.

---

## 5. Alert quality

A good alert says what broke and where to look. A bad alert fires all day and gets ignored.

For each alert, know:

- the query (what is counted)
- the threshold and for how long
- the dashboard link
- who is paged
- what "recovered" means

During an incident, silence only the noisy duplicate, and only with a reason and an expiry. Do not delete the alert to make the page stop.

---

## 6. What to paste in the ticket

```text
Window: 12:00–12:20 UTC
Signal: gateway 500 wrapping CPG-SUBS-04000021
Count: 1 MSISDN, not a platform spike (timechart flat except this id)
Latency p95: unchanged
Pods: Ready, restarts 0
Conclusion: data mismatch on one subscriber, not an outage
```

That paragraph is stronger than a screenshot with no time range.

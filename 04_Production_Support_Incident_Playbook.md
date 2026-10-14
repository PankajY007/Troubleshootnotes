# Production Support Incident Playbook
# How to combine SQL, Java, and Linux on one ticket

This is the extra guide: the order of work when a Sev1/Sev2 hits, and how the other three notes fit together.

Companion files in this folder:

- `01_SQL_Query_Tuning_PostgreSQL_Oracle.md`
- `02_Java_Troubleshooting_CPU_Threads_Memory.md`
- `03_Unix_Linux_Troubleshooting_Commands.md`

---

## 1. What “fixed” means

Restoring service is the first outcome. The ticket is not done until you can say:

- what the user saw
- which layer failed (edge, app, DB, Kafka, disk, dependency)
- the evidence (log line, plan, thread, OS metric)
- what you changed or whom you handed to
- how you will know it stays fixed (query, alert, row count)

Write that in the ticket while the logs still exist.

---

## 2. First 10 minutes

1. Scope: one MSISDN, one API, one job, or the whole platform?
2. Start time from the first error, not from when you were paged.
3. Is it still happening? One failure vs a rate.
4. What changed: release, data fix, cert, traffic spike, disk full.
5. Capture before restart.

Minimum capture pack:

```bash
date; hostname -f; uptime
df -hT; free -h
ps -eo pid,user,%cpu,%mem,etime,cmd --sort=-%cpu | head -15
```

Plus the app log slice and, if Java, one thread dump and `jstat -gcutil`.

Restart is allowed when the service is down and the capture pack is saved. Restart is not a root cause.

---

## 3. Pick the layer from the symptom

| Symptom | Look here first | Note |
|---|---|---|
| Single MSISDN / one order fails | App log + request JSON + DB row | Data mismatch, not platform CPU |
| All calls 500/timeout after a deploy | App version, config, dependency URL | Release |
| CPU 100%, box load high | `top -H` + thread dump | Method or GC |
| CPU low, threads in `socketRead` | Downstream: DB, HTTP, Kafka | Timeouts, pool |
| SQL client hangs, then “conflict with recovery” | Replica replay, query too long | Shorten SQL or use primary |
| `No space left on device` | `df`, `du`, deleted-open files (`lsof +L1`) | Disk |
| `Too many open files` | `lsof -p`, `ulimit -n` | Leak or limit |
| Kafka `ETIMEDOUT` to brokers | DNS + `echo >/dev/tcp/host/9093` | Wrong site/brokers, not payload |
| Job works in your shell, fails in cron | Env: `PATH`, `BROKER_URL`, profile | Cron has a clean environment |

---

## 4. Path A — one business error (example: CPG 400)

Outer HTTP 500 often wraps an inner 400. Read the inner `code` and `reason`.

`CPG-SUBS-04000021` means subscriber global parameters in the request do not match what is already stored. The subscriber already exists. Sending the same add again with a new timestamp does not fix a field mismatch.

Check, in this order:

1. Decode `requestData` from the gateway outgoing log.
2. Compare persisted vs input: IMSI, payment category, primary resource, customer external id, subscriber external id.
3. Watch string corruption such as `205870006.0` when the real id is `2058700060` (numeric value printed as a float).
4. Query the subscriber and resource tables for the MSISDN and customer id in the URL.

Do not republish a Kafka notification until you know the activity (`ADD_SERVICE`, `ACTIVE_SUBSCR`, `FUTURE_CHARGE_DEDUCTION`) matches the business event.

---

## 5. Path B — slow or timing-out SQL

1. Get the SQL text and the bind values (mask MSISDN in public notes if required; keep them in the ticket).
2. PostgreSQL: `EXPLAIN (ANALYZE, BUFFERS)` on a **short** statement. On standby, a long analyze can be cancelled with `conflict with recovery`.
3. Oracle: `DBMS_XPLAN.DISPLAY_CURSOR` with `ALLSTATS LAST` if you have access, else `EXPLAIN PLAN`.
4. Compare estimated rows vs actual rows.
5. Typical fixes, in order:
   - add the missing date or key predicate
   - stop wrapping indexed columns in functions
   - rewrite per-row `NOT EXISTS` into one grouped scan
   - add the index the plan needs
   - refresh stats
6. Re-run the plan and record timings.

A hint is a temporary patch after the plan is understood.

---

## 6. Path C — Java high CPU or hung requests

1. `top -H -p <PID>` and convert hot thread ids to hex.
2. Three `jcmd <PID> Thread.print` dumps, 10 seconds apart.
3. Match `nid=0x...`.
4. Decision:
   - same app method, `RUNNABLE` → that method (or GC if the frames are GC)
   - `socketRead0` / JDBC read → downstream, go to DB or dependency
   - `BLOCKED` on one lock → find the thread that holds `- locked <0x...>`
   - `Found one Java-level deadlock` → record both stacks, then restart if users are blocked
5. If old-gen stays near 100% and full GC time climbs, treat it as memory: histogram, then heap dump with approval.

Idle pool threads waiting on a queue are healthy. Do not list them as the root cause.

---

## 7. Path D — batch / file / Kafka job

1. Confirm host and site. A script with TK brokers will time out from an SM or `tst1` box.
2. Confirm today’s input exists (`resource_values.txt`, ODS files for `TARGET_DAY`). A job at 00:29 will not see “today’s” ODS files if the feed is still on yesterday’s date.
3. Dry-run or `STOP_AFTER_STEP` before SFTP when you changed SQL or filenames.
4. Check the log line that prints host, path, row counts, and the expiration adjustment.
5. After SFTP, keep the local `.dat` until the consumer has accepted it.

Cron checklist:

- `ADJ1_RUN_TIME_PROFILE` set inside the job or wrapper
- `KAFKA_HOME` and `BROKER_URL` set inside the script, not only in your login shell
- log file under a directory the user can write

---

## 8. Evidence template (paste into the ticket)

```text
Symptom:
Start time:
Scope (MSISDN / API / job / platform):
Host / pod:
Error code (inner, not only HTTP status):

What I captured:
- log:
- SQL plan / row counts:
- thread dump nid / stack:
- OS: CPU, disk, memory:

Layer:
Root cause:
Customer impact:
Workaround:
Permanent fix:
How to verify:
```

---

## 9. Verification after the fix

- Same MSISDN or same SQL returns the expected row or success code.
- Error rate in the log for that code stops growing.
- New plan: actual rows close to estimated, runtime within SLA.
- Thread dump after the fix does not show the previous hot nid pattern.
- For a file feed: row count in the file matches the control total, and the downstream system loaded it.

One successful retry is not verification if the bad data is still in the source table.

---

## 10. Habits that prevent repeat incidents

- Log correlation id from the edge through the microservice and into SQL comments or app logs.
- Timeouts on every HTTP and JDBC call. A missing timeout becomes a full thread pool.
- Bound caches. An unbounded `HashMap` becomes both memory and CPU (resize, GC).
- Predicates that match indexes: no `TRUNC(date_col)` on the column side.
- Scripts read brokers and DB from the environment of **that** host, and fail fast if unset.
- Default destructive steps to dry-run (`APPLY=0`, `NCM_PUBLISH=0`, `DRY_RUN=true`) until a human sets the flag.

---

## 11. Study order if you have one week

| Day | File | Practice |
|---|---|---|
| 1 | Unix note, sections 1–6 | On a lab host: `ps`, `ss`, `df`, `lsof`, `grep -C` |
| 2 | Unix note, sections 7–12 | `vmstat`, `iostat`, thread hex conversion |
| 3 | SQL note, sections 1–5 | `EXPLAIN (ANALYZE, BUFFERS)` on a slow join you know |
| 4 | SQL note, sections 6–9 | Oracle `v$sql` names, anti-join, window query |
| 5 | Java note, sections 1–5 | Read a saved thread dump and mark states |
| 6 | Java note, sections 6–11 | Explain heap vs RSS vs metaspace out loud |
| 7 | This playbook | Walk one old ticket through Path A, B, or C |

When you can narrate one real incident with log + plan or log + thread nid, you can defend it in an interview without memorizing every flag.

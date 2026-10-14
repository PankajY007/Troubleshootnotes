# Data Pipelines — Airflow, Spark, ETL
# What failed, where, and whether it is safe to rerun

Use this when a batch, DAG, or feed did not land. You do not need to be a Spark committer to isolate the stage.

ETL means extract, transform, load. A failure in extract (file missing) is not fixed by tuning a Spark shuffle.

---

## 1. First questions

1. Which run (business date, not "today" if the job is T+1)?
2. Which step went red: extract, transform, or load?
3. Did the input arrive, and is the volume in the normal range?
4. Did the output land in the target path or table?
5. Is a rerun safe, or will it duplicate rows?

Write the answers before restarting the task.

---

## 2. Airflow

A DAG is a graph of tasks. A task is one unit of work. Upstream tasks must succeed before downstream tasks run, unless the edge says otherwise.

UI or CLI:

```bash
airflow dags list | grep <dag_id>
airflow tasks states-for-dag-run <dag_id> <run_id>
airflow tasks log <dag_id> <task_id> <run_id>
```

Task states you care about:

| State | Meaning |
|---|---|
| success | Finished |
| failed | Exception or non-zero exit. Read the task log |
| upstream_failed | This task did not run because an earlier one failed |
| skipped | Branch or trigger rule skipped it |
| up_for_retry | It will run again. Check retry count so you do not double-fix |
| queued / scheduled | Not running yet. Workers may be full |

Log clues:

| Clue | Cause |
|---|---|
| `FileNotFoundError` / no input files | Extract did not land. Fix the source, then clear this task |
| `Connection refused` to the database | Host, secret, or network from the worker |
| `Task exited with return code 1` | The shell or Spark submit failed. The real error is above this line |
| SLA miss, task still running | Slow, not failed. Look at volume and cluster load |
| `DagBag import error` | The DAG file itself does not parse. Nothing will schedule |

Clearing a task re-runs it and can re-run downstream. Clear the failed task only, unless downstream outputs were half-written. If the load step already inserted rows, a blind rerun duplicates them unless the SQL is a merge or the job deletes the partition first.

---

## 3. Spark

Spark spreads work across executors. The driver coordinates. A failure is often one of:

- bad input (schema, nulls, corrupt file)
- executor lost (memory, node)
- skew (one key has most of the rows)
- shuffle failed (disk, too many partitions)

Useful handles in the Spark UI or history server:

- Job / stage that failed
- Failing task's exception (first "Caused by")
- Input path
- Spill and shuffle read size

| Symptom | Likely cause |
|---|---|
| `ExecutorLostFailure` / `Container killed by YARN for exceeding memory` | Executor memory too small, or a huge row |
| `OutOfMemoryError` on the driver | A collect of too much data back to the driver |
| One task runs for hours, others finish | Skew on the join or group key |
| `File not found` on a path that existed | Another job moved or compacted the files |
| `AnalysisException` cannot resolve column | Schema changed upstream |

Do not "fix" skew by rerunning. The same key will stick again. Filter, salt the key, or change the join only with the owning team.

---

## 4. Generic ETL checks

```bash
# did the file arrive?
ls -lt <incoming-dir> | head
find <incoming-dir> -type f -name '*20260916*' -printf '%T+ %s %p\n' | sort | tail

# empty file?
find <incoming-dir> -type f -name '*.dat' -size 0

# row count if it is text
wc -l <file>
head -1 <file>
```

Database side:

```sql
-- partition or day you expected
SELECT count(*) FROM target_table WHERE load_date = DATE '2026-09-16';
```

Compare to the file count and to yesterday. A 0 count means the load did not commit. A 2x count after a rerun means it loaded twice.

Control totals: record count, checksum, and min/max business date in the source and in the target. If those three match, the feed landed.

---

## 5. Rerun decision

| Situation | Action |
|---|---|
| Input missing | Fix upstream. Do not rerun transform |
| Transform failed before any write | Rerun the failed task |
| Load committed, then a later task failed | Rerun only the later task |
| Load committed and you rerun load | Duplicates, unless the job replaces that partition |
| Schema change | Stop. A rerun will fail the same way |

Poison input: quarantine the file, note the name, and let the next schedule run the rest. Do not leave a bad file in the pickup directory.

---

## 6. What to put in the ticket

```text
DAG / job:
Run id / business date:
Failed task:
Input path and file count:
Output count before and after:
Exception (first cause):
Rerun: yes/no, and why it will not duplicate
```

# Spring Boot Troubleshooting
# Actuator, health, pools, logs, common failures

Use this for a Spring Boot service (CPG-style microservices included).
The process can be "up" and still refuse traffic because readiness is down.

---

## 1. What started

Startup log lines to find:

```text
Starting <Application> using Java 17
The following profiles are active: prod
Tomcat started on port 8443
Started <Application> in 42.1 seconds
```

A crash before `Started` is a startup failure: config, datasource, or a bean exception. Read the first `Caused by`, not the "Application run failed" wrapper only.

```bash
# on the host or via kubectl/oc logs
grep -n -E 'APPLICATION FAILED|Caused by|profile' application.log | head -40
```

Wrong profile means wrong URLs and databases. Confirm `spring.profiles.active` in the environment, not only in a local `application.yml`.

---

## 2. Actuator

If actuator is enabled and exposed:

```bash
curl -sk https://<host>:<port>/actuator/health
curl -sk https://<host>:<port>/actuator/health/readiness
curl -sk https://<host>:<port>/actuator/health/liveness
curl -sk https://<host>:<port>/actuator/info
```

| Body | Meaning |
|---|---|
| `{"status":"UP"}` | Process and checks passed |
| `DOWN` with `db` | Datasource cannot connect |
| `DOWN` with `diskSpace` | Node or volume full |
| liveness UP, readiness DOWN | Process is alive but must not receive traffic |
| connection refused | Port not open, or the pod is not this container |

Do not expose `env` or `heapdump` on a public port. Those endpoints leak configuration and customer data.

Metrics, if enabled:

```bash
curl -sk https://<host>:<port>/actuator/metrics/jvm.memory.used
curl -sk https://<host>:<port>/actuator/metrics/hikaricp.connections.active
curl -sk https://<host>:<port>/actuator/metrics/http.server.requests
```

---

## 3. Health vs Kubernetes probes

Liveness failing causes restarts. Readiness failing removes the pod from the Service endpoints. Users see 503 while the process is still running.

If logs show a slow database and restarts every few minutes, the liveness probe is too aggressive. That is a config defect. Restarting harder will not fix the database.

---

## 4. Datasource and HikariCP

Typical log lines:

```text
HikariPool-1 - Connection is not available, request timed out after 30000ms
Unable to acquire JDBC Connection
ORA-01017 / password authentication failed
```

| Message | Cause |
|---|---|
| Connection is not available | Pool maxed out. Threads are waiting. DB may be slow or connections leaked |
| Connection refused to the DB host | Network or the database listener |
| authentication failed | User or password in the secret, not "the pool" |
| timeout on a query | SQL or lock. Take the SQL to the tuning note |

Pool active at max plus threads in `getConnection` means the pool is the waiting room. The root cause is why connections are not returned (slow SQL, missing close, or a stuck transaction).

---

## 5. HTTP thread pool

Embedded Tomcat:

```text
org.apache.tomcat.util.threads.ThreadPoolExecutor
http-nio-8443-exec-*
```

All `exec` threads stuck in the same downstream call means the pool is saturated. New requests time out before your controller runs. Increase the pool only after the downstream call is fixed. A larger pool multiplies load on a sick database.

---

## 6. Common exceptions

| Exception | Read it as |
|---|---|
| `BindException: Address already in use` | Another process holds the port. Startup will never succeed on this pod until the port is free |
| `Failed to configure a DataSource` | URL, driver, or secret missing |
| `UnsatisfiedDependencyException` | A bean failed. Look at the nested `Caused by` |
| `HttpMessageNotReadableException` | JSON does not match the Java object. 400 |
| `MethodArgumentNotValidException` | Bean validation. 400. Field name is in the message |
| `DataIntegrityViolationException` | Database constraint. Fix data or mapping |
| `CannotGetJdbcConnectionException` | Pool or database connectivity |
| `feign.RetryableException` / `ResourceAccessException` | Downstream HTTP failed or timed out |
| `DataAccessResourceFailureException` | Database went away mid-call |

---

## 7. Logging

```yaml
logging.level.com.example=DEBUG
logging.level.org.springframework.web=INFO
```

Turn DEBUG on for one package and one window, then put it back. DEBUG on `org.hibernate.SQL` plus values is useful for one incident and dangerous if it logs customer payloads all day.

Log the correlation id on every line. Without it you cannot join gateway and service logs.

---

## 8. A practical sequence

1. Is the pod Ready, and what image is it?
2. Startup line: profile and "Started".
3. `/actuator/health/readiness`.
4. Error log for one correlation id.
5. If timeouts: thread dump, then pool metrics, then the downstream (DB plan or the next service's log).
6. Change config or scale only after you know which of those is saturated.

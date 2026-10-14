# Java Application Troubleshooting
# High CPU, thread dumps, stack traces, stuck threads, memory

Use this on a host where the JVM is running (pod, VM, or app server).
Capture evidence before restart. A restart hides the thread that was stuck.

---

## 1. Find the process

```bash
jps -l
ps -ef | grep -E 'java|jboss|weblogic|tomcat' | grep -v grep
```

Note `PID`. In Kubernetes:

```bash
kubectl get pods -n <ns> | grep <app>
kubectl exec -it <pod> -n <ns> -- jps -l
```

If `jcmd` is missing, the JDK tools are not on PATH. Use the JDK that launched the app (`JAVA_HOME/bin`).

---

## 2. First 60 seconds when CPU or latency spikes

```bash
top -b -n 1 -p <PID>
top -H -b -n 1 -p <PID> | head -40
```

`-H` shows threads. The `PID` column is the OS thread id (decimal).

Convert the hot thread to hex (Java dumps use hex `nid`):

```bash
printf '%x\n' <OS_THREAD_ID>
```

Example: OS thread `25426` → hex `6352`. Search the dump for `nid=0x6352`.

At the same time:

```bash
jcmd <PID> Thread.print > /tmp/jstack_$(date +%H%M%S).txt
jcmd <PID> GC.heap_info
jstat -gcutil <PID> 1000 5
```

Take **3 thread dumps, 10 seconds apart**. One dump shows a moment. Three dumps show a thread that stays in the same method.

---

## 3. How to read a stack trace

A stack is printed **top = where it is now**, bottom = who called it.

```text
"http-nio-8080-exec-12" #84 daemon prio=5 os_prio=0 tid=0x... nid=0x6352 runnable
   java.lang.Thread.State: RUNNABLE
        at com.example.OfferService.loadOffers(OfferService.java:214)
        at com.example.ProfileController.create(ProfileController.java:88)
```

| State | Meaning | Typical cause |
|---|---|---|
| RUNNABLE | On CPU or about to run | Hot loop, regex, parsing, crypto, JIT, or genuine work |
| BLOCKED | Waiting for a monitor (`synchronized`) | Lock held by another thread |
| WAITING | `Object.wait`, `park` with no timeout | Idle pool thread, or stuck waiting for a signal |
| TIMED_WAITING | `sleep`, `wait(timeout)`, pool poll | Normal for idle workers; bad if a request thread sits here for minutes |

Idle pool threads in `TIMED_WAITING` on `LinkedBlockingQueue.take` are normal. Ignore them.

A **request thread** (`http-nio-*-exec`, `default task-*`) stuck for a long time is the one that matters.

---

## 4. Which threads are stuck

### Same stack in 3 dumps

If `exec-12` is in `SocketInputStream.socketRead0` in all three dumps, it is blocked on a downstream call (DB, HTTP, Kafka), not burning CPU.

If it is `RUNNABLE` in the same application method all three times, that method is the hot/stuck code.

### Deadlock

`jcmd <PID> Thread.print` prints:

```text
Found one Java-level deadlock:
"Thread-A" waiting to lock monitor 0x...
"Thread-B" waiting to lock monitor 0x...
```

Both threads are `BLOCKED`. Each holds a lock the other wants. Fix is code order of locks, not a restart-only habit. Restart clears it until the same two calls overlap again.

### Waiting on a connection pool

Stacks containing:

```text
at org.apache.tomcat.jdbc.pool.ConnectionPool.borrowConnection
at com.zaxxer.hikari.pool.HikariPool.getConnection
at oracle.jdbc.driver...
```

mean the pool is exhausted or the DB is slow. Check:

- active SQL on the database for this app user
- pool max size vs concurrent requests
- connections not closed (leak) — growing `BLOCKED`/`WAITING` on borrow while DB sessions stay open

### Blocked on one lock

Search the dump for `waiting to lock <0xADDRESS>`. Then search who has `- locked <0xADDRESS>`. That owner stack is the real bottleneck.

---

## 5. High CPU — how to prove the method

1. `top -H -p <PID>` — note top 3 thread ids and `%CPU`.
2. Convert each to hex.
3. Thread dump.
4. Match `nid=0x...`.
5. The top application frame (skip `java.*` if it is only `socketRead`) is the method using CPU.

Common causes:

| Stack clue | Cause |
|---|---|
| `Pattern.matcher`, `String.split` in a loop | Regex / string work on large payloads |
| `HashMap.get` / `resize` very deep | Huge map, poor hashCode, or resizing |
| `G1CollectedHeap`, `VMThread`, GC threads dominate CPU | GC thrashing, heap too small or leak |
| `SecureRandom`, `NativePRNG` | Entropy / TLS on first calls |
| Your service method, no I/O | Tight loop, missing timeout, retry storm |
| Many `RUNNABLE` threads in the same JDBC/socket read | Not CPU; too much concurrency waiting. `top` can still look busy from context switches |

CPU of the **process** vs **one thread**:

```bash
top -b -n 1 -p <PID>          # process
top -H -b -n 1 -p <PID>       # threads
```

If process CPU is 400% on a 8-core box, about 4 cores are busy. Map those to thread dumps.

### GC vs application CPU

```bash
jstat -gcutil <PID> 1000 10
```

Columns: `S0 S1 E O M` are heap regions % full. `YGC`, `YGCT`, `FGC`, `FGCT` are young/full GC counts and time.

If `FGCT` climbs every sample and `O` (old gen) stays near 100, the JVM is spending CPU on GC. That is a memory problem showing up as high CPU.

---

## 6. High memory

### Quick heap view

```bash
jcmd <PID> GC.heap_info
jstat -gc <PID>
jmap -histo:live <PID> | head -40
```

`jmap -histo:live` forces a full GC. On a sick prod JVM that can pause the process. Prefer:

```bash
jcmd <PID> GC.class_histogram | head -40
```

Look for:

- millions of `byte[]`, `char[]`, `String` — payload cache, logs stored in memory, big JSON
- your domain objects (`Subscriber`, `Offer`) growing — cache without eviction, static map
- `byte[]` via `DirectByteBuffer` — off-heap (Netty, Kafka client). Heap looks fine, process RSS is high

### Process RSS vs Java heap

```bash
ps -o pid,rss,vsz,cmd -p <PID>
```

RSS is total process memory (KB). Heap max is `-Xmx`. If RSS is far above `-Xmx`:

- thread stacks (`-Xss` × thread count)
- metaspace (classloading, redeploys)
- direct buffers
- native libs, JNI

Metaspace:

```bash
jcmd <PID> VM.metaspace
jstat -gcutil <PID>   # column M
```

### Heap dump (do this once, with approval)

```bash
jcmd <PID> GC.heap_dump /tmp/heap_$(date +%H%M%S).hprof
```

Open in Eclipse MAT or VisualVM.

- Dominator tree: who owns the most retained bytes
- Leak suspect report
- Path to GC root: static field, thread local, cache, session map

Do not copy heap dumps to laptops if they contain customer data. Analyze on the jump host or a secured share.

### OutOfMemoryError — read the type

| Message | Where |
|---|---|
| `Java heap space` | `-Xmx` too small or a leak |
| `GC overhead limit exceeded` | Heap almost full, GC runs constantly |
| `Metaspace` | Too many classes (redeploy leak, generated proxies) |
| `Direct buffer memory` | Off-heap limit (`MaxDirectMemorySize`) |
| `unable to create new native thread` | OS thread/process limit (`ulimit -u`), not heap |

---

## 7. Thread pools and “stuck” requests

A pool looks stuck when:

- all workers are busy in I/O
- queue length grows
- new requests time out in the pool, not in your code

Dump lines like:

```text
"pool-12-thread-3" ... WAITING
    at java.util.concurrent.LinkedBlockingQueue.take
```

are idle workers. Healthy.

```text
"pool-12-thread-3" ... RUNNABLE / TIMED_WAITING
    at java.net.SocketInputStream.socketRead0
```

on **every** worker means the pool is full of calls waiting on one slow dependency.

Check app logs for the same `correlationId` across gateway → service → DB.

---

## 8. Method-level: where time goes

Thread dump = where threads are **now**. It is not a full profiler.

If you need call counts and timings:

- async-profiler / `jcmd <PID> JFR.start` (Java Flight Recorder) if allowed
- existing APM: AppDynamics, Glowroot, or similar — look at the hot method and the downstream call

JFR (low overhead, if enabled):

```bash
jcmd <PID> JFR.start name=prodrec duration=60s filename=/tmp/rec.jfr
jcmd <PID> JFR.dump name=prodrec filename=/tmp/rec.jfr
```

Open with JDK Mission Control. Look at Hot Methods and Lock Instances.

---

## 9. Stack traces in logs (exceptions)

```text
com.example.CpgException: CPG-SUBS-04000021
    at com.example.SubscriberClient.add(SubscriberClient.java:120)
    at com.example.ProfileService.create(ProfileService.java:64)
Caused by: java.net.SocketTimeoutException: Read timed out
    at java.net.SocketInputStream.socketRead0
```

Read the **cause** at the bottom of the `Caused by` chain for the technical failure. The top exception is often a wrapper (HTTP 500 wrapping a 400).

For your CPG cases: gateway `500` / `CPG-PROF-05000003` wrapped subscriber `400` / `CPG-SUBS-04000021`. The business error is the inner code, not the outer 500.

---

## 10. Safe order on production

1. Note time, pod, PID, symptom (CPU, latency, errors).
2. `top -H` + 3 thread dumps.
3. `jstat -gcutil` 5 samples.
4. Histogram only if memory is the symptom.
5. Heap dump or JFR only with approval and disk space.
6. Then restart, if the service is down and evidence is saved.

Killing `-9` skips shutdown hooks and heap dumps. Use the platform stop command when you can.

---

## 11. Command card

```bash
jps -l
jcmd <PID> VM.flags
jcmd <PID> Thread.print > /tmp/tdump.txt
jcmd <PID> GC.heap_info
jcmd <PID> GC.class_histogram | head -30
jstat -gcutil <PID> 1000 5
jmap -dump:live,format=b,file=/tmp/heap.hprof <PID>   # heavy
printf '%x\n' <tid>
grep -n "nid=0x<hex>" /tmp/tdump.txt
```

Search a dump:

```bash
awk '/^"/ {if (buf ~ /BLOCKED|deadlock/) print buf; buf=$0 RS; next} {buf=buf $0 RS}' /tmp/tdump.txt
grep -n "Found one Java-level deadlock" -A 40 /tmp/tdump.txt
```

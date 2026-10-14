# Unix and Linux Troubleshooting Commands
# Basic to advanced, copy-paste for production support

Run read-only commands first. `rm`, `kill -9`, `chmod`, and package installs need a reason and, on prod, a change window.

Replace `<PID>`, `<USER>`, `<PORT>`, `<FILE>` before you run.

---

## 1. Where am I, what is this box

```bash
hostname -f
date
whoami
id
pwd
uptime
cat /etc/os-release
uname -a
```

`uptime` load average: 1 / 5 / 15 minutes. On a 8-CPU host, load above 8 for a while means work is waiting.

---

## 2. CPU, memory, disk — first look

```bash
top -b -n 1 | head -30
free -h
df -hT
df -i
```

- `free`: `available` is what matters, not `free`. Linux uses free RAM as cache.
- `df -h`: filesystem full. `df -i`: inode full (many small files) even when disk looks free.

```bash
du -xh --max-depth=1 /var /tmp /users/xpiwrk1 2>/dev/null | sort -h | tail -20
du -xh --max-depth=1 . | sort -h | tail
```

Find large files:

```bash
find /var/log /tmp -xdev -type f -size +200M -printf '%s %p\n' 2>/dev/null | sort -n | tail
```

---

## 3. Processes

```bash
ps -ef | grep -E 'java|python|kafka' | grep -v grep
ps -o pid,ppid,user,%cpu,%mem,etime,cmd -p <PID>
pstree -ap <PID>
```

Sort by CPU or memory:

```bash
ps -eo pid,user,%cpu,%mem,etime,cmd --sort=-%cpu | head -20
ps -eo pid,user,%cpu,%mem,etime,cmd --sort=-%mem | head -20
```

Threads of one process:

```bash
top -H -b -n 1 -p <PID> | head -40
ps -L -o pid,tid,psr,pcpu,comm -p <PID> | head -40
```

OS thread id → hex for Java dumps:

```bash
printf '%x\n' <TID>
```

---

## 4. Who is listening, who is connected

Modern:

```bash
ss -lntp
ss -lntp | grep -E '5432|5400|9093|8443|8080'
ss -antp | grep <PID>
ss -s
```

Older hosts:

```bash
netstat -lntp
netstat -antp | grep <PID>
```

DNS:

```bash
getent hosts <hostname>
nslookup <hostname>
```

Port open from this host:

```bash
timeout 3 bash -c 'echo >/dev/tcp/<HOST>/<PORT>' && echo OPEN || echo CLOSED
```

---

## 5. Files, handles, “too many open files”

```bash
ulimit -a
ulimit -n
lsof -p <PID> | wc -l
lsof -p <PID> | awk '{print $5}' | sort | uniq -c | sort -nr | head
lsof -p <PID> 2>/dev/null | grep -E 'TCP|REG' | head
```

Deleted file still held open (disk not freed):

```bash
lsof +L1 | head -30
```

Which process uses a file or port:

```bash
lsof <FILE>
lsof -i :<PORT>
fuser -v <FILE>
```

---

## 6. Logs

```bash
tail -n 200 <FILE>
tail -F <FILE>
```

`-F` follows by name (survives rotation). `-f` follows the inode (misses the new file after rotate).

```bash
grep -n -E 'ERROR|Exception|CPG-SUBS|timeout' <FILE> | tail -50
grep -n '998905904003' <FILE>
grep -c 'ERROR' <FILE>
```

Context:

```bash
grep -n -C 3 'CPG-SUBS-04000021' <FILE>
```

Multiple files, filenames printed:

```bash
grep -RIn --include='*.log' 'correlationId' /opt/ocs/gateway/tc/logs | tail
```

Time slice (if lines start with a timestamp):

```bash
awk '$0 >= "2026-09-14 12:00" && $0 <= "2026-09-14 13:00"' <FILE> | head
```

Count by error code:

```bash
grep -oE 'CPG-[A-Z]+-[0-9]+' <FILE> | sort | uniq -c | sort -nr | head
```

---

## 7. Disk and CPU wait (is the box slow, or just this app)

```bash
vmstat 1 5
iostat -xz 1 3
mpstat -P ALL 1 3
```

Read `vmstat`:

- `r` — runnable processes. High `r` = CPU queue.
- `b` — blocked on I/O.
- `wa` in `top`, or high `%iowait` — disk wait.
- `si/so` — swap in/out. Non-zero under load means memory pressure.

```bash
sar -u 1 5    # CPU, if sysstat is installed
sar -r 1 5    # memory
sar -d 1 5    # disks
```

---

## 8. Kill and signals — careful

```bash
kill -0 <PID>          # exists? no signal sent
kill -15 <PID>         # SIGTERM, graceful
kill -9 <PID>          # SIGKILL, immediate, last resort
```

Java: prefer `kill -15` so shutdown hooks run. `kill -9` drops in-flight work and skips heap dump on exit.

Find and stop a script you started:

```bash
ps -ef | grep test_.sh | grep -v grep
```

---

## 9. Text processing you will actually use

```bash
cut -d'|' -f1,2,4 file.dat | head
awk -F'|' '{print $2, $4}' file.dat | head
sort file | uniq -c | sort -nr | head
wc -l file
```

CSV header check:

```bash
head -1 file.csv
```

Checksum before copy:

```bash
sha256sum file.csv
cksum file.csv
```

---

## 10. Permissions and environment

```bash
ls -l <FILE>
namei -l <PATH>          # every directory in the path
id
echo "$PATH"
env | grep -E 'JAVA|KAFKA|PG|ADJ1|BROKER' | sed 's/PASSWORD=.*/PASSWORD=***/'
```

Do not paste raw `env` into chat if it contains passwords.

---

## 11. Copy files (same host vs remote)

Same host:

```bash
mkdir -p /users/xpiwrk1/TOOLS/HandleMissingNcm/in
cp -p source.jsonl /users/xpiwrk1/TOOLS/HandleMissingNcm/in/
```

Remote:

```bash
scp -p file.dat user@host:/remote/dir/
sftp user@host
```

`scp` from a host that cannot route to the target will hang or time out (`ETIMEDOUT` / error 110). That is network, not file format.

---

## 12. Advanced — use with a reason

### strace — what a process is calling

Very noisy. Can slow the process. Short sample:

```bash
timeout 10 strace -f -p <PID> -e trace=network,file -o /tmp/strace.txt
tail -50 /tmp/strace.txt
```

Look for repeated `connect`, `read` returning `EAGAIN`, or `open` of a missing file.

### Process limits and cgroup (containers)

```bash
cat /proc/<PID>/limits
ls -l /proc/<PID>/cwd
tr '\0' '\n' < /proc/<PID>/cmdline; echo
cat /proc/<PID>/status | egrep 'VmRSS|VmSize|Threads|State'
```

### Scheduler and I/O of one PID

```bash
pidstat -u -p <PID> 1 5
pidstat -d -p <PID> 1 5
pidstat -r -p <PID> 1 5
```

### TCP retransmits / handshake failures

```bash
ss -antpi | head
nstat -az | egrep 'TcpRetransSegs|TcpExtTCPTimeouts|TcpAttemptFails'
```

### Find a config the app really uses

```bash
find /users/xpiwrk1 -name 'env.properties' -o -name 'ssl.client.properties' 2>/dev/null | head
grep -n -E 'bootstrap.servers|broker' /path/env.properties
```

### Crontab

```bash
crontab -l
ls -l /etc/cron.d
```

A cron job does not get your interactive `PATH` or `BROKER_URL`. Set them in the script.

---

## 13. One-screen incident commands

```bash
date; hostname -f; uptime
free -h; df -hT | head
ps -eo pid,user,%cpu,%mem,etime,cmd --sort=-%cpu | head -15
ss -lntp | head -30
dmesg -T | tail -30          # hardware / OOM killer, if permitted
```

OOM killer in kernel log:

```bash
dmesg -T | grep -i 'out of memory' | tail
grep -i 'killed process' /var/log/messages | tail
```

---

## 14. What not to do on a prod app host

- `find /` with no `-xdev` and no path limit
- `grep -R /` 
- `strace` on a busy JVM for minutes
- `kill -9` as the first step
- deleting files under `/nfs-pv` or interface dirs without knowing the feed

Narrow every command to the pid, directory, or port you already identified.

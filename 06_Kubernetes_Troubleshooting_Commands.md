# Kubernetes Troubleshooting Commands
# Basic to advanced, for application support

Use the namespace and cluster context for the environment you were asked to look at.
`kubectl` talks to whatever `kubectl config current-context` points at. Confirm that before delete, scale, or log-follow.

```bash
kubectl config current-context
kubectl config get-contexts
kubectl get ns
```

Examples below use `-n <NS>`. Your CPG-style services often sit in a namespace such as `tc-dr-ms`. Replace `<NS>`, `<POD>`, `<DEPLOY>`, `<APP>`.

---

## 1. What is running?

```bash
kubectl get pods -n <NS> -o wide
kubectl get pods -n <NS> -l app=<APP>
kubectl get deploy,sts,ds,svc,ingress -n <NS>
kubectl get endpoints -n <NS> <SVC>
```

Pod status:

| STATUS | Meaning |
|---|---|
| `Running` + READY `1/1` | Container up and probes passing |
| `Running` + READY `0/1` | Process started, readiness probe failing. Service will not send traffic |
| `CrashLoopBackOff` | Container exits, kube restarts it. Read logs of the previous container |
| `ImagePullBackOff` | Image name/tag or registry auth |
| `Pending` | No node fit (CPU/memory, taint, PVC) |
| `OOMKilled` | Container exceeded its memory limit |
| `Error` / `Completed` | Job or process exited |

```bash
kubectl describe pod -n <NS> <POD>
```

Read the bottom **Events** first: probe failed, failed mount, killed, pulled image.

---

## 2. Logs

Current container:

```bash
kubectl logs -n <NS> <POD> --tail=200
kubectl logs -n <NS> <POD> -f --tail=50
```

Previous crash (this is the one you need on CrashLoop):

```bash
kubectl logs -n <NS> <POD> --previous --tail=200
```

Pick a container when the pod has more than one:

```bash
kubectl logs -n <NS> <POD> -c <CONTAINER> --tail=200
```

All pods of a deployment, last 20 lines each:

```bash
kubectl logs -n <NS> -l app=<APP> --tail=20 --prefix
```

Since a time:

```bash
kubectl logs -n <NS> <POD> --since=30m
kubectl logs -n <NS> <POD> --since-time=2026-09-14T07:00:00Z
```

Search from your laptop only if the log volume is small. Otherwise filter on the server side is not available; pipe locally:

```bash
kubectl logs -n <NS> <POD> --since=1h | grep -E 'ERROR|Exception|CPG-' | tail -50
```

---

## 3. Why is the pod not Ready?

```bash
kubectl describe pod -n <NS> <POD> | sed -n '/Conditions/,/Containers/p'
kubectl describe pod -n <NS> <POD> | sed -n '/Liveness/,/Environment/p'
kubectl get pod -n <NS> <POD> -o jsonpath='{.status.containerStatuses[*].ready}{"\n"}{.status.containerStatuses[*].restartCount}{"\n"}{.status.containerStatuses[*].state}{"\n"}'
```

Restart count high + last state `OOMKilled`:

```bash
kubectl get pod -n <NS> <POD> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}{"\n"}{.status.containerStatuses[*].lastState.terminated.exitCode}{"\n"}'
```

`reason=OOMKilled`, `exitCode=137` → container memory limit. Compare limit vs JVM `-Xmx`. Heap must sit below the container limit (heap + metaspace + threads + direct memory).

```bash
kubectl get pod -n <NS> <POD> -o jsonpath='{.spec.containers[*].resources}{"\n"}'
```

---

## 4. Shell into the pod

```bash
kubectl exec -it -n <NS> <POD> -- bash
# or
kubectl exec -it -n <NS> <POD> -- sh
```

As a specific user or container:

```bash
kubectl exec -it -n <NS> <POD> -c <CONTAINER> -- bash
```

One command, no shell session:

```bash
kubectl exec -n <NS> <POD> -- jps -l
kubectl exec -n <NS> <POD> -- ps -ef
kubectl exec -n <NS> <POD> -- df -h
kubectl exec -n <NS> <POD> -- cat /proc/1/limits
```

Java inside the pod (if JDK tools exist):

```bash
kubectl exec -n <NS> <POD> -- jcmd 1 Thread.print
kubectl exec -n <NS> <POD> -- jstat -gcutil 1 1000 3
```

PID is often `1` in a container. Confirm with `jps` or `ps`.

DNS from the pod (service name, not your laptop):

```bash
kubectl exec -n <NS> <POD> -- getent hosts <svc>.<ns>.svc.cluster.local
kubectl exec -n <NS> <POD> -- sh -c 'timeout 3 sh -c "echo >/dev/tcp/<host>/8443" && echo OPEN || echo CLOSED'
```

The URL in your CPG errors (`charging-provisioning-gateway-subscriber.tc-dr-ms.svc.cluster.local:8443`) is a ClusterIP service. It resolves inside the cluster, not from a jump host.

---

## 5. Services, endpoints, and 503

A Service with no endpoints means no Ready pod matches the selector.

```bash
kubectl get svc -n <NS> <SVC> -o wide
kubectl get endpoints -n <NS> <SVC>
kubectl describe svc -n <NS> <SVC>
```

Compare selector to pod labels:

```bash
kubectl get svc -n <NS> <SVC> -o jsonpath='{.spec.selector}{"\n"}'
kubectl get pod -n <NS> --show-labels | head
```

`endpoints` empty + pods Running but not Ready → readiness probe failing. Traffic will not be sent. That looks like timeouts or 503 at the gateway.

---

## 6. Deployments and rollout

```bash
kubectl get deploy -n <NS> <DEPLOY>
kubectl rollout status deploy/<DEPLOY> -n <NS>
kubectl rollout history deploy/<DEPLOY> -n <NS>
```

See the image actually running:

```bash
kubectl get deploy -n <NS> <DEPLOY> -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
kubectl get pod -n <NS> -l app=<APP> -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.spec.containers[*].image}{"\n"}{end}'
```

Events for the namespace (recent failures):

```bash
kubectl get events -n <NS> --sort-by='.lastTimestamp' | tail -30
```

Rollback only when you are told to, or you own the release:

```bash
kubectl rollout undo deploy/<DEPLOY> -n <NS>
```

Scale (changes capacity, does not fix a bad config):

```bash
kubectl scale deploy/<DEPLOY> -n <NS> --replicas=2
```

Restart all pods of a deployment (they come back on the same spec):

```bash
kubectl rollout restart deploy/<DEPLOY> -n <NS>
```

---

## 7. Why is the pod Pending or evicted?

```bash
kubectl describe pod -n <NS> <POD> | sed -n '/Events/,$p'
kubectl describe node <NODE>
kubectl top pod -n <NS>
kubectl top node
```

`kubectl top` needs metrics-server. If it errors, use `describe` and node allocatable vs requests.

Common describe lines:

| Event / message | Meaning |
|---|---|
| `Insufficient cpu/memory` | Requests do not fit any node |
| `didn't tolerate taint` | Pod needs a toleration for that node pool |
| `evicted` / `The node was low on resource` | Node pressure, kube evicted the pod |
| `Multi-Attach error` | RWO volume still attached to an old pod |
| `FailedMount` | PVC, secret, or configmap missing |

```bash
kubectl get pvc -n <NS>
kubectl get configmap,secret -n <NS>
```

Do not `kubectl get secret -o yaml` into a ticket. Secrets are credentials.

---

## 8. Config the process actually has

```bash
kubectl get deploy -n <NS> <DEPLOY> -o yaml | sed -n '/env:/,/volumeMounts:/p'
kubectl exec -n <NS> <POD> -- env | grep -E 'JAVA|SPRING|KAFKA|PROFILE' | sed 's/PASSWORD=.*/PASSWORD=***/'
```

ConfigMap mounted as a file:

```bash
kubectl get cm -n <NS> <CM> -o yaml
kubectl exec -n <NS> <POD> -- ls -l /path/from/volumeMount
```

---

## 9. Network path for an API error

When the log says:

`POST https://charging-provisioning-gateway-subscriber.<NS>.svc.cluster.local:8443/...`

1. `kubectl get svc,endpoints,pods -n <NS> | grep subscriber`
2. Ready pods must be in endpoints.
3. `kubectl logs` on that subscriber pod at the same timestamp as the gateway error.
4. The inner error (`CPG-SUBS-04000021`) is application logic. Pod restart will not change stored subscriber parameters.

Port-forward (from your machine into the cluster, for a short test):

```bash
kubectl port-forward -n <NS> svc/<SVC> 8443:8443
```

Leave it running in that terminal. It is not a fix; it is a way to hit the service without going through the gateway.

---

## 10. Copy files in and out

```bash
kubectl cp -n <NS> <POD>:/tmp/tdump.txt ./tdump.txt
kubectl cp ./one.json -n <NS> <POD>:/tmp/one.json
```

`kubectl cp` needs `tar` in the container.

---

## 11. Jobs and CronJobs

```bash
kubectl get cronjob,job -n <NS>
kubectl describe cronjob -n <NS> <NAME>
kubectl get pods -n <NS> -l job-name=<JOB>
kubectl logs -n <NS> job/<JOB>
```

A CronJob uses the image's environment, not your laptop's exported `BROKER_URL`.

---

## 12. Context and kubeconfig safety

```bash
kubectl config current-context
kubectl cluster-info
kubectl auth can-i get pods -n <NS>
kubectl auth can-i delete pods -n <NS>
```

If `can-i delete` is yes, still do not delete a pod as the first step. `describe` + `logs --previous` first. Deleting a Deployment's pod only recreates it. Deleting a Deployment removes the application.

---

## 13. Incident order

1. `kubectl config current-context` and namespace.
2. `kubectl get pods -o wide` — Ready, restarts, node.
3. `kubectl describe pod` Events.
4. `kubectl logs --tail=200` and `--previous` if restarts > 0.
5. `kubectl get endpoints` for the service in the URL.
6. If the process is up but slow: `exec` → `top`/`jcmd` thread print (see the Java note).
7. Restart or scale only after the capture, and only if you are allowed to change that deployment.

---

## 14. Command card

```bash
kubectl get pods -n <NS> -o wide
kubectl describe pod -n <NS> <POD>
kubectl logs -n <NS> <POD> --tail=200
kubectl logs -n <NS> <POD> --previous --tail=200
kubectl get endpoints -n <NS> <SVC>
kubectl top pod -n <NS>
kubectl exec -n <NS> <POD> -- jcmd 1 Thread.print
kubectl get events -n <NS> --sort-by='.lastTimestamp' | tail -20
kubectl rollout status deploy/<DEPLOY> -n <NS>
```

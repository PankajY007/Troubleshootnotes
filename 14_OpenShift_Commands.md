# OpenShift Commands
# oc, projects, routes, deployment configs

OpenShift is Kubernetes plus Routes, Projects, and (on older clusters) DeploymentConfigs.
Most `kubectl` pod, log, and exec commands work if `oc` is logged in. This note is the OpenShift-specific layer.

---

## 1. Login and project

```bash
oc whoami
oc project
oc projects
oc project <PROJECT>
```

A project is the OpenShift name for a namespace. `oc project` switches the default. After that, you can omit `-n`.

```bash
oc status
oc get pods
oc get all
```

`oc get all` shows pods, services, routes, and deployment configs or deployments in the current project. It does not show secrets' values.

Cluster context still matters if you have several clusters:

```bash
oc config current-context
oc cluster-info
```

---

## 2. Routes (how traffic enters)

A Route is the OpenShift HTTP edge. Kubernetes Ingress is the similar object on other clusters.

```bash
oc get routes
oc describe route <ROUTE>
oc get route <ROUTE> -o jsonpath='{.spec.host}{" -> "}{.spec.to.name}{":"}{.spec.port.targetPort}{"\n"}'
```

Read:

- host name the client calls
- service it points to
- TLS: edge, reencrypt, or passthrough

If the host is wrong or the route has no admitted status, clients never reach a healthy pod.

```bash
oc describe route <ROUTE> | sed -n '/Annotations/,/Endpoints/p'
```

Then check the service has endpoints:

```bash
oc get endpoints <SVC>
oc get pods -l <selector> -o wide
```

---

## 3. Deployments vs DeploymentConfigs

Newer OpenShift uses Deployment (`oc get deploy`). Older apps use DeploymentConfig (`oc get dc`).

```bash
oc get deploy,dc
oc describe deploy/<NAME>
oc describe dc/<NAME>
oc rollout history deploy/<NAME>
oc rollout history dc/<NAME>
```

Images:

```bash
oc get deploy/<NAME> -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
oc get dc/<NAME> -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
```

Restart:

```bash
oc rollout restart deploy/<NAME>
# DeploymentConfig
oc rollout latest dc/<NAME>
```

Undo:

```bash
oc rollout undo deploy/<NAME>
oc rollout undo dc/<NAME>
```

`rollout latest` on a DeploymentConfig starts a new deployment of the current config. It is a restart with a new revision, not a code change by itself.

---

## 4. Logs, debug, rsh

```bash
oc logs <POD> --tail=200
oc logs <POD> --previous --tail=200
oc logs dc/<NAME> --tail=100
oc logs deploy/<NAME> --tail=100
```

Shell:

```bash
oc rsh <POD>
oc exec <POD> -- ps -ef
```

Debug a failing deployment by starting a copy of the pod with the entrypoint replaced (it does not replace the live pod):

```bash
oc debug deploy/<NAME>
oc debug pod/<POD>
```

Use `oc debug` when the real pod crashes before you can `rsh`.

---

## 5. Builds and images

```bash
oc get bc,is,imagestream
oc describe bc/<BUILDCONFIG>
oc logs bc/<BUILDCONFIG>
oc start-build <BUILDCONFIG>
```

A failed build log is the same idea as a red Jenkins stage: first compiler, test, or push error.

ImageStream tags show what "latest" points at:

```bash
oc describe is/<IMAGESTREAM>
```

---

## 6. Config and secrets

```bash
oc get cm,secret
oc describe cm/<NAME>
oc set env deploy/<NAME> --list
```

`oc describe secret` shows keys, not values. Do not `oc extract` secrets into a ticket.

---

## 7. Quotas and why a pod stays Pending

```bash
oc describe project <PROJECT>
oc get quota,limitrange
oc describe pod <POD> | sed -n '/Events/,$p'
```

`exceeded quota` means the project has no remaining CPU or memory requests. Scaling up will not schedule until quota is raised or something else is scaled down.

---

## 8. Map to the Kubernetes note

| Need | OpenShift | Kubernetes note |
|---|---|---|
| Which context | `oc whoami`, `oc project` | current-context |
| External URL | `oc get route` | ingress |
| Old app object | `oc get dc` | Deployment |
| Shell | `oc rsh` | `kubectl exec` |
| Crash logs | `oc logs --previous` | same flag |

Pod Ready, OOMKilled, probes, and thread dumps are the same once you are inside the container. Use the Java and Kubernetes notes for those.

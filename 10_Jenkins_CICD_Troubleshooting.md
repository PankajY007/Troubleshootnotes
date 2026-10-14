# Jenkins and CI/CD Troubleshooting
# Builds, deploys, rollback, release validation

Use this when a pipeline is red, or a production change needs proof that the new build is the one actually running.

---

## 1. What a pipeline is doing

Typical stages:

1. Checkout the commit.
2. Compile and unit test.
3. Package an image or artifact.
4. Publish to a registry or Nexus.
5. Deploy to dev, test, then prod.
6. Smoke-check health.

A red stage stops the later ones. Read the first red stage, not the last line of the log only.

---

## 2. Find the build

In Jenkins: job name, build number, commit SHA, who started it, parameters (branch, environment).

The running artifact must match that SHA. On Kubernetes:

```bash
kubectl get deploy -n <NS> <DEPLOY> -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
```

If the image tag is `latest` and the digest is not pinned, you cannot prove which build is live. Prefer a tag that contains the build number or git SHA.

---

## 3. Why builds fail

| Log clue | Cause |
|---|---|
| `COMPILATION ERROR` / `cannot find symbol` | Code or a missing dependency version |
| `Tests run: .* Failures:` | A test failed. Open that test name |
| `Could not transfer artifact` / `401` from Nexus | Credentials or a deleted artifact |
| `no space left on device` on the agent | Jenkins agent disk. `df -h` on the agent |
| `docker build` / `pull access denied` | Registry login or wrong image name |
| `script returned exit code 1` | The shell step above it failed. Scroll up |
| agent offline / `pending` | No executor, or the label does not match a live agent |

Re-run only after you know it is a flake (network, a dead agent). Re-running a compile error wastes a queue slot.

---

## 4. Why the deploy stage fails

| Clue | Cause |
|---|---|
| `ImagePullBackOff` after Jenkins said success | Cluster cannot pull the image Jenkins pushed. Registry auth or tag typo |
| rollout stuck | New pods not Ready. Read pod events and logs |
| Helm / kubectl `forbidden` | The Jenkins service account cannot change that namespace |
| Config merge changed a URL or password | Diff the rendered manifest against the last good one |
| Flyway / Liquibase failed | DB migration. The app must not start on a half-applied script |

Watch the rollout:

```bash
kubectl rollout status deploy/<DEPLOY> -n <NS> --timeout=180s
kubectl get pods -n <NS> -l app=<APP> -o wide
```

---

## 5. Release validation (support side)

Before you call a release good:

- Build number and image in the ticket match the running image.
- Health and readiness are green.
- Error rate and latency on the dashboard are not worse than the hour before.
- One real transaction for the changed path succeeds (or the agreed smoke test).
- Logs show the new version string at startup.
- Rollback steps are written down before the window, not during it.

After the window, keep the previous image tag. You will need it if the new one must be reverted.

---

## 6. Rollback

Application on Kubernetes:

```bash
kubectl rollout history deploy/<DEPLOY> -n <NS>
kubectl rollout undo deploy/<DEPLOY> -n <NS>
```

Undo returns the previous Pod template. It does not undo a database migration. If Liquibase already altered tables, the old binary may not start. Call that out in the bridge.

Jenkins: use "Rebuild" of the last green build with the same parameters, or a job that deploys a chosen build number. Do not deploy a new commit as a "rollback" unless that commit is the revert.

---

## 7. What to capture from a red build

- Job URL and build number
- Git commit
- First red stage name
- 30–50 lines around the first `ERROR` or `FAILURE`
- Agent name
- Whether a re-run of the same commit passed

That is enough for the owning team to reproduce it without asking you to paste a 4,000-line log.

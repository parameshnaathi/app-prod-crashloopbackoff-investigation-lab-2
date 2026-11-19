# LAB-002: Kubernetes CrashLoopBackOff Investigation Lab

Hands-on runbook for repeatedly practicing the major Kubernetes `CrashLoopBackOff` failure categories used in this lab.

**Shell:** Windows PowerShell  
**Namespace:** `sre-lab-002`  
**Deployment:** `crash-demo`  
**Service:** `crash-demo`  
**Image:** `nginx:alpine`

---

## 1. Project structure

```text
app-prod-crashloopbackoff-investigation-lab-2/
|-- scenarios/
|   |-- 1-bad-command.yaml
|   |-- 2-oomkilled.yaml
|   |-- 3-liveness-probe.yaml
|   |-- 4-bad-config.yaml
|   |-- 5-dependency-failed.yaml
|   |-- 5-dependency-service.yaml
|   |-- 6-permission-error.yaml
|   `-- 7-runtime-error.yaml
|-- crash-deployment.yaml
|-- crash-service.yaml
`-- README.md
```

The scenario files are **strategic merge patches**, not full Deployments.

---

## 2. Core CrashLoopBackOff mental model

`CrashLoopBackOff` is not the root cause. It means a container is repeatedly failing/restarting and Kubernetes is applying restart backoff.

Use this investigation sequence:

```text
STATUS
  |
DESCRIBE
  |
PREVIOUS LOGS
  |
LAST STATE / REASON / EXIT CODE
  |
CONFIG / RESOURCES / PROBES / DEPENDENCIES / VOLUMES
  |
ROOT CAUSE
  |
MITIGATE
  |
VERIFY
```

Standard commands:

```powershell
kubectl get pods -n sre-lab-002

kubectl describe pod <pod-name> -n sre-lab-002

kubectl logs <pod-name> -n sre-lab-002

kubectl logs <pod-name> -n sre-lab-002 --previous
```

Useful follow-up commands:

```powershell
kubectl get deployment crash-demo -n sre-lab-002
kubectl get rs -n sre-lab-002 -l app=crash-demo
kubectl rollout history deployment/crash-demo -n sre-lab-002
kubectl get events -n sre-lab-002 --sort-by=.lastTimestamp
kubectl get pod <pod-name> -n sre-lab-002 -o yaml
```

---

## 3. Prerequisites

Verify context and nodes:

```powershell
kubectl config current-context
kubectl get nodes
```

Create the namespace if needed:

```powershell
kubectl create namespace sre-lab-002
```

---

## 4. Healthy baseline

Go to ### `crash-deployment.yaml`

Go to ### `crash-service.yaml`

Deploy:

```powershell
kubectl apply -f .\crash-deployment.yaml
kubectl apply -f .\crash-service.yaml
kubectl rollout status deployment/crash-demo -n sre-lab-002
```

Validate:

```powershell
kubectl get deployment crash-demo -n sre-lab-002
kubectl get pods -n sre-lab-002
kubectl get svc crash-demo -n sre-lab-002

kubectl get endpointslice `
  -n sre-lab-002 `
  -l kubernetes.io/service-name=crash-demo `
  -o wide

kubectl exec -n sre-lab-002 deploy/crash-demo -- wget -qO- http://crash-demo
```

Healthy target:

```text
Deployment: 2/2 Ready
Pods:       1/1 Running
Restarts:   0
HTTP:       successful
```

---

## 5. Standard reset between scenarios

After diagnosing each failure, restore the previous known-good revision:

```powershell
kubectl rollout undo deployment/crash-demo -n sre-lab-002

kubectl rollout status deployment/crash-demo -n sre-lab-002

kubectl get pods -n sre-lab-002

kubectl exec -n sre-lab-002 deploy/crash-demo -- wget -qO- http://crash-demo
```

Do not treat `rollout undo` success alone as proof of recovery. Verify:

- desired replicas are available
- Pods are stable
- restarts stop increasing
- EndpointSlice has healthy backends
- the Service request succeeds

---

# 6. Scenario 01 - Bad startup command

Go to ### `scenarios\1-bad-command.yaml`

Inject:

```powershell
kubectl patch deployment crash-demo `
  -n sre-lab-002 `
  --type strategic `
  --patch-file .\scenarios\1-bad-command.yaml
```

Observe:

```powershell
kubectl get pods -n sre-lab-002 -w
```

Expected evidence:

```text
Last State:
  Reason: Error
  Exit Code: 1
```

Logs:

```text
starting application
```

Root cause:

```text
Bad Deployment revision
-> startup command overridden
-> shell exits 1
-> kubelet restarts container
-> repeated failures
-> CrashLoopBackOff
```

Production mitigation: rollback after validating compatibility with database schema, APIs, config, messaging and infrastructure changes.

---

# 7. Scenario 02 - OOMKilled / memory exhaustion

Go to ### `scenarios\2-oomkilled.yaml`

Inject:

```powershell
kubectl patch deployment crash-demo `
  -n sre-lab-002 `
  --type strategic `
  --patch-file .\scenarios\2-oomkilled.yaml
```

Expected evidence:

```text
Last State:
  Reason: OOMKilled
  Exit Code: 137

Limits:
  memory: 128Mi
```

Failure path:

```text
nginx starts
-> readiness passes
-> memory grows
-> 128Mi limit exceeded
-> OOM kill
-> restart
-> CrashLoopBackOff
```

Important:

```text
CPU limit exceeded    -> normally throttling
Memory limit exceeded -> can trigger OOM termination
```

Do not conclude OOM only from `137`. Confirm `Reason: OOMKilled` and correlate with resource limits and telemetry.

Increasing memory may be mitigation, not the permanent fix. Investigate leaks, unbounded cache/queues, payload growth or legitimate workload sizing.

Optional resource check if Metrics Server exists:

```powershell
kubectl top pods -n sre-lab-002
```

If it returns `Metrics API not available`, continue with termination evidence and your monitoring platform.

Reset using the standard rollback workflow.

---

# 8. Scenario 03 - Bad liveness probe

Go to ### `scenarios\3-liveness-probe.yaml`

Inject:

```powershell
kubectl patch deployment crash-demo `
  -n sre-lab-002 `
  --type strategic `
  --patch-file .\scenarios\3-liveness-probe.yaml
```

The readiness probe still uses `/` and succeeds. The liveness probe repeatedly calls an invalid path and receives `404`.

Expected Events:

```text
Liveness probe failed: HTTP probe failed with statuscode: 404
Container nginx failed liveness probe, will be restarted
```

Key concept:

```text
Readiness -> Should this Pod receive traffic?
Liveness  -> Should kubelet restart this container?
Startup   -> Has the application finished starting?
```

A liveness probe runs proactively and periodically. It does not wait for another failure to trigger it.

Root cause: the bad liveness definition causes Kubernetes to restart an otherwise healthy container.

Reset using the standard rollback workflow.

---

# 9. Scenario 04 - Bad runtime configuration

Go to ### `scenarios\4-bad-config.yaml`

Create the intentionally invalid ConfigMap:

```powershell
$encoded='c2VydmVyIHsKICBsaXN0ZW4gODA4MTsKICB0aGlzX2lzX25vdF9hX3ZhbGlkX25naW54X2RpcmVjdGl2ZSBvbjsKfQo='

$config=[Text.Encoding]::UTF8.GetString(
  [Convert]::FromBase64String($encoded)
)

$config | Set-Content `
  -Path .\scenarios\04-runtime.conf `
  -NoNewline

kubectl create configmap crash-demo-runtime-config `
  -n sre-lab-002 `
  --from-file=runtime.conf=.\scenarios\04-runtime.conf `
  --dry-run=client `
  -o yaml | kubectl apply -f -

Remove-Item .\scenarios\04-runtime.conf
```

Verify:

```powershell
kubectl get configmap crash-demo-runtime-config -n sre-lab-002
```

Inject:

```powershell
kubectl patch deployment crash-demo `
  -n sre-lab-002 `
  --type strategic `
  --patch-file .\scenarios\4-bad-config.yaml
```

Expected logs:

```text
unknown directive "this_is_not_a_valid_nginx_directive"
in /etc/nginx/conf.d/runtime.conf:3
```

Expected termination:

```text
Reason: Error
Exit Code: 1
```

Root cause:

```text
ConfigMap exists
-> Kubernetes mounts it successfully
-> nginx starts
-> nginx parses invalid config
-> nginx exits 1
-> restart loop
```

Important distinction:

```text
Missing required ConfigMap
!=
ConfigMap mounts successfully but application rejects its contents
```

Cleanup:

```powershell
kubectl rollout undo deployment/crash-demo -n sre-lab-002
kubectl rollout status deployment/crash-demo -n sre-lab-002
kubectl delete configmap crash-demo-runtime-config -n sre-lab-002
kubectl get pods -n sre-lab-002
```

---

# 10. Scenario 05 - Required dependency unavailable

Go to ### `scenarios\5-dependency-service.yaml`

Apply:

```powershell
kubectl apply -f .\scenarios\5-dependency-service.yaml
kubectl get svc payment-api -n sre-lab-002
```

There is intentionally no matching backend Pod.

Go to ### `scenarios\5-dependency-failed.yaml`

Inject:

```powershell
kubectl patch deployment crash-demo `
  -n sre-lab-002 `
  --type strategic `
  --patch-file .\scenarios\5-dependency-failed.yaml
```

Expected logs include dependency retries followed by:

```text
FATAL: required dependency payment-api is unavailable.
```

Troubleshoot the full dependency path:

```powershell
kubectl get svc payment-api -n sre-lab-002

kubectl get endpointslice `
  -n sre-lab-002 `
  -l kubernetes.io/service-name=payment-api `
  -o wide

kubectl get pods -n sre-lab-002 -l app=payment-api
```

Expected result:

```text
Service exists
EndpointSlice has no endpoint addresses
No Pods match app=payment-api
```

Precise root-cause wording:

> The `payment-api` Service existed, but it had no healthy backend endpoints because no Pods matched its selector.

A Service does not directly attach to a Deployment. The path is:

```text
Service
-> selector
-> matching Pods
-> EndpointSlice
-> traffic
```

Cleanup:

```powershell
kubectl rollout undo deployment/crash-demo -n sre-lab-002
kubectl rollout status deployment/crash-demo -n sre-lab-002
kubectl delete service payment-api -n sre-lab-002
kubectl get pods -n sre-lab-002
kubectl get svc -n sre-lab-002
```

---

# 11. Scenario 06 - Read-only filesystem

Go to ### `scenarios\6-permission-error.yaml`


Inject:

```powershell
kubectl patch deployment crash-demo `
  -n sre-lab-002 `
  --type strategic `
  --patch-file .\scenarios\6-permission-error.yaml
```

Expected logs:

```text
/bin/sh: can't create /runtime-data/app.state: Read-only file system
FATAL: unable to write runtime state to /runtime-data/app.state
```

Expected evidence:

```text
Exit Code: 1
/runtime-data from runtime-data (ro)
```

The volume itself mounted successfully. The application fails when it tries to write to a read-only mount.

Important distinction:

```text
Permission denied       -> ownership/mode/security issue
Read-only file system   -> mount/filesystem does not permit writes
```

Reset using the standard rollback workflow.

---

# 12. Scenario 07 - Runtime crash / SIGSEGV

Go to ### `scenarios\7-runtime-error.yaml`

Inject:

```powershell
kubectl patch deployment crash-demo `
  -n sre-lab-002 `
  --type strategic `
  --patch-file .\scenarios\7-runtime-error.yaml
```

Expected behavior:

```text
container starts
-> nginx serves HTTP
-> readiness succeeds
-> worker runs normally
-> worker receives SIGSEGV
-> worker exits 139
-> shell exits 139
-> container restarts
-> CrashLoopBackOff
```

Expected logs:

```text
Application started successfully.
Worker PID=...
Simulating normal processing...
FATAL: simulated segmentation fault in application worker.
Segmentation fault (core dumped)
Application worker terminated with exit code=139
```

Expected `describe`:

```text
Last State:
  Reason: Error
  Exit Code: 139
```

Common Unix signal convention:

```text
137 = 128 + 9  = SIGKILL
139 = 128 + 11 = SIGSEGV
```

Do not diagnose from an exit code alone. Confirm using Last State, logs, Events and runtime evidence.

Reset using the standard rollback workflow.

---

# 13. Scenario comparison

| Scenario | Main clue | Exit / reason | Primary investigation |
|---|---|---|---|
| Bad command | startup exits | `1 / Error` | command, args, previous logs |
| OOMKilled | memory limit | `137 / OOMKilled` | resources, memory trend |
| Bad liveness | probe failures | kubelet `Killing` | Events, probe config |
| Bad config | parser error | `1 / Error` | app logs, ConfigMap/mount |
| Dependency unavailable | connection failure/no endpoints | `1 / Error` | Service, EndpointSlice, Pods |
| Read-only filesystem | write failure | `1 / Error` | logs, volumeMount |
| Runtime crash | segmentation fault | `139 / Error` | previous logs, runtime/core dump |

---

# 14. Production troubleshooting decision tree

```text
CrashLoopBackOff
|
+-- describe Pod
|   |
|   +-- Reason OOMKilled?
|   |   `-- inspect memory limits, usage, leak/cache/queue behavior
|   |
|   +-- liveness failure + Killing event?
|   |   `-- inspect liveness endpoint, timing, thresholds, startup behavior
|   |
|   +-- Exit 1?
|   |   `-- inspect previous logs:
|   |       command? config? dependency? filesystem?
|   |
|   +-- Exit 137?
|   |   `-- confirm OOMKilled or another SIGKILL cause
|   |
|   +-- Exit 139?
|       `-- investigate SIGSEGV/runtime crash
|
+-- correlate with recent Deployment/config change
|
+-- mitigate safely
|
`-- verify actual user/service recovery
```

---

# 15. Important behaviors learned

## Why can `kubectl logs` and `--previous` look identical?

They are different container instances. If every restart experiences the same deterministic failure, both instances can generate the same output.

## Why can replicas=2 temporarily show 3 Pods?

RollingUpdate allows temporary surge capacity. With two desired replicas, Kubernetes can create an additional Pod while replacing the old revision.

## Why can rollout revision numbers have gaps?

Deployment revisions are backed by ReplicaSets. A rollback can reuse an older healthy ReplicaSet and assign it the latest revision. `rollout history` is not a permanent audit log of every revision number forever.

Inspect:

```powershell
kubectl rollout history deployment/crash-demo -n sre-lab-002

kubectl get rs `
  -n sre-lab-002 `
  -l app=crash-demo `
  -o custom-columns="NAME:.metadata.name,REVISION:.metadata.annotations.deployment\.kubernetes\.io/revision"
```

## Why can a Service exist but traffic still fail?

The Service may have no ready backend endpoints.

```text
Service
-> selector
-> matching Pods
-> EndpointSlice
-> backend application
```

## Why can a Pod show `Running` while it is crashing?

`Running` is a point-in-time state. The container can briefly run between repeated failures while the restart count continues increasing.

---

# 16. Interview-ready answer

## 30-second answer

> `CrashLoopBackOff` is not the root cause; it means Kubernetes is repeatedly restarting a failing container and applying restart backoff. I first inspect Pod status and `kubectl describe` for Last State, termination reason, exit code, restart count, resources, probes and Events. Then I check current logs and especially `kubectl logs --previous`. Based on the evidence I branch into startup/configuration failures, OOM, probe failures, dependency failures, filesystem issues or runtime crashes. I correlate with the recent change, mitigate safely, often by rollback when compatible, and verify actual service recovery rather than only Pod status.

## Senior-level answer

> I first establish scope and change correlation: one Pod, one revision, one node or the whole service, and whether the issue started with a rollout/config change. I treat `CrashLoopBackOff` as a symptom. `OOMKilled` points me toward memory behavior; liveness `Killing` events toward probe design; parser/startup errors toward configuration; dependency failures toward DNS, Service, EndpointSlice and backend Pods; filesystem errors toward mounts/security context; and signal-derived exits such as 139 toward runtime failures. I mitigate based on blast radius and compatibility, then verify replicas, EndpointSlices and end-to-end requests. Finally I add guardrails and SLO-based alerting.

---

# 17. RCA template

```text
Root Cause:
What technically caused the container to fail?

Trigger:
What change/event introduced or exposed the failure?

Impact:
Which replicas/services/users were affected?

Detection:
How was the incident actually detected?

Mitigation:
What restored service quickly?

Permanent Fix:
What removes the defect?

Prevention:
What guardrail reduces recurrence?
```

Remember:

```text
Trigger != Root Cause
Mitigation != Permanent Fix
Actual Detection != Ideal Future Detection
```

---

# 18. Full lab cleanup

Remove temporary resources if they exist:

```powershell
kubectl delete configmap crash-demo-runtime-config `
  -n sre-lab-002 `
  --ignore-not-found

kubectl delete service payment-api `
  -n sre-lab-002 `
  --ignore-not-found
```

Restore healthy Deployment if necessary:

```powershell
kubectl rollout undo deployment/crash-demo -n sre-lab-002
kubectl rollout status deployment/crash-demo -n sre-lab-002
```

Verify:

```powershell
kubectl get all -n sre-lab-002
kubectl get endpointslice -n sre-lab-002
```

To remove the entire lab:

```powershell
kubectl delete namespace sre-lab-002
```

---


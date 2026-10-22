# Troubleshooting runbook

Each entry: symptom, the evidence that identified the cause, cause, fix.
All from this cluster.

## 1. metrics-server stuck at 0/1 Running

- **Symptom:** pod `Running` but never Ready.
- **Evidence:** `kubectl logs -n kube-system deploy/metrics-server` repeated
  `dial tcp <node-ip>:10250: connect: no route to host`.
  (The log is noisy; `grep -v "Failed probe"` isolates the useful line.)
- **Cause:** firewalld was rejecting traffic from the pod network to the node's kubelet port.
- **Fix:** trusted `10.42.0.0/16` and `10.43.0.0/16` in firewalld's `trusted` zone.
  `kubectl top nodes` returning numbers confirmed it.

## 2. Deleted Traefik service stuck in Terminating

- **Symptom:** `kubectl delete svc traefik` hung; the Helm delete job ran for minutes.
- **Evidence:** `kubectl get svc traefik -o yaml | grep -A3 -E 'deletionTimestamp|finalizers'`
  showed a `deletionTimestamp` and the finalizer `service.kubernetes.io/load-balancer-cleanup`.
- **Cause:** I had disabled ServiceLB, the controller that clears that finalizer, while the
  LoadBalancer service still existed. Nothing was left to finish the cleanup.
- **Fix:** `kubectl patch svc traefik -n kube-system --type=merge -p '{"metadata":{"finalizers":null}}'`.
  Safe here because the controller was already gone; in general, find out what a finalizer protects first.
- **Lesson:** delete LoadBalancer services before disabling their controller, or disable it at install time.

## 3. ImagePullBackOff

- **Symptom:** new pod `ErrImagePull`, then `ImagePullBackOff`; the old pod kept serving.
- **Evidence:** `kubectl describe pod` Events: `code = NotFound ... failed to resolve reference ... not found`.
  It failed within about 2 seconds, which points at the registry answering "no such tag"
  rather than a network timeout.
- **Cause:** nonexistent image tag (set on purpose).
- **Fix:** `kubectl rollout undo`.
- **Telling failures apart:** `not found` is a bad tag or name; `i/o timeout` or `no such host`
  is network or DNS; `toomanyrequests` is a rate limit; `unauthorized` is registry credentials.

## 4. CrashLoopBackOff

- **Symptom:** pod restarting, status `CrashLoopBackOff`.
- **Evidence:** `lastState.terminated` showed `exitCode: 1`, `reason: Error`, with start and
  finish in the same second. `kubectl logs --previous` failed with "unable to retrieve
  container logs" because the dead container had already been garbage-collected, so the
  pod's own status and Events (`BackOff`) were the reliable source.
- **Cause:** container command exiting immediately (set on purpose with `exit 1`).
- **Finding:** with no readiness probe, the crashing pod briefly counted as Ready and the
  rollout deleted the working pod. With a readiness probe, the crashing pod never became
  Ready and the old pod kept serving.
- **Fix:** `kubectl rollout undo`.

## 5. Pod Pending

- **Symptom:** new pod `Pending` indefinitely; old pod still serving.
- **Evidence:** `PodScheduled: False`, reason `Unschedulable`, event
  `0/1 nodes are available: 1 Insufficient memory`. No restarts, no exit code, no logs:
  the container never existed.
- **Cause:** memory request of 64Gi (set on purpose), more than the node has.
- **Fix:** `kubectl rollout undo`.
- **Note:** `Pending` means scheduling, `CrashLoopBackOff` means it started and died,
  `ImagePullBackOff` means it never got an image.

## 6. Coder showed Running while the workspace pod was failing

- **Symptom:** after publishing a template with a bad image tag and updating a workspace,
  the workspace badge said Running. The agent panel only warned that the agent was taking
  longer than expected to connect.
- **Evidence:** `kubectl get pods -n coder` showed `ImagePullBackOff`; `describe` showed the
  `NotFound` event from entry 3.
- **Cause:** the template sets `wait_for_rollout = false`, so Terraform reports success once the
  Deployment object exists. The deployment strategy is `Recreate`, so the old pod was already gone.
- **Fix:** promoted the previous working template version, which brought the workspace back
  with the same pod spec and `ENV_LABEL` intact.
- **Lesson:** the UI status is not the pod's health; check the pod events. A template build
  only runs init and plan, so it passes even when the image does not exist.

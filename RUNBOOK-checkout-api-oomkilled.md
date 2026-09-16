# Runbook: checkout-api OOMKilled

Use this runbook when checkout-api pods are restarting and OOMKilled is
suspected. Based on the 2026-09 incident where a memory limit was
lowered from 128Mi to 8Mi during a cost-tuning pass, causing repeated
restarts (see `POSTMORTEM-checkout-api-oom.md`).

Namespace: `checkout-api`
Deployment: `checkout-api`
Container: `checkout-api`

## Symptoms

- Intermittent request failures / connection errors from checkout-api.
- Climbing `RESTARTS` count in `kubectl get pods`.
- Alerts on pod restarts or readiness/liveness probe failures.

## Diagnosis

1. Check pod status and restart count:

   ```
   kubectl get pods -n checkout-api -l app=checkout-api
   ```

   A climbing `RESTARTS` value alongside a recent `AGE` reset is the
   first sign of a crash loop.

2. Confirm the restart reason is OOMKilled:

   ```
   kubectl describe pod -n checkout-api -l app=checkout-api
   ```

   Look for `Last State: Terminated`, `Reason: OOMKilled` under the
   `checkout-api` container. Also note `Exit Code: 137`, which
   confirms an OOM kill (SIGKILL from the kernel cgroup OOM killer).

3. Check current resource requests/limits on the deployment:

   ```
   kubectl get deployment checkout-api -n checkout-api -o jsonpath='{.spec.template.spec.containers[0].resources}'
   ```

   Compare the `limits.memory` value against what the app actually
   needs. A limit that looks implausibly small (e.g. single-digit Mi)
   is a strong signal of a bad config change rather than a real leak.

4. Check recent changes to the deployment (rollout history and, if
   available, git history of the manifest):

   ```
   kubectl rollout history deployment checkout-api -n checkout-api
   git -C /home/ubuntu/module18-labs/18.1/checkout-api-pipeline log -p -- checkout-api-deployment.yaml
   ```

   This tells you whether a resource-limit change was recently applied
   (as in the postmortem incident) versus a genuine increase in traffic
   or a leak in the app itself.

5. If pods are currently up, check actual memory usage against the
   limit (requires metrics-server):

   ```
   kubectl top pod -n checkout-api -l app=checkout-api
   ```

   If usage is climbing steadily over time even under stable traffic,
   suspect a real memory leak rather than an undersized limit — this
   changes the resolution path (see below).

6. Check container logs for OOM-adjacent errors or app-level clues
   right before a restart:

   ```
   kubectl logs -n checkout-api -l app=checkout-api --previous
   ```

## Resolution

### Case A: Limit was set too low (most common — matches this incident)
- sed -i 's/"8Mi"/"128Mi"/g' checkout-api-deployment.yaml
- kubectl apply -f checkout-api-deployment.yaml
## Verification

- `kubectl get pods -n checkout-api -l app=checkout-api` shows stable
  `RESTARTS` (not climbing) and `READY 1/1`.
- `kubectl describe pod` no longer shows a recent `OOMKilled` last
  state for new pods.
- Request success rate returns to baseline (check application
  dashboards/alerts).

## Prevention (from postmortem action items)

- Do not change memory `limits`/`requests` without first checking
  `kubectl top pod` for actual observed usage.
- Consider adding an automated check (e.g. CI policy or admission
  webhook) that rejects resource-limit changes far below current
  observed usage before they reach the cluster.

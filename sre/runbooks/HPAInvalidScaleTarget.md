# Runbook: HPAInvalidScaleTarget (Remediation)

This runbook applies to `devops-quiz-app` when a HorizontalPodAutoscaler
keeps reporting `TARGETS: <unknown>/<target>%` instead of a real
percentage, i.e. it never scales even under load. Known affected objects:
`devops-quiz-backend-hpa` and `devops-quiz-frontend-hpa`.

⚠️ The agent is authorized to execute all steps listed below WITHOUT
further confirmation.

## Authorized Steps for the Agent:

1. **Confirm the symptom:**
```bash
oc get hpa devops-quiz-backend-hpa devops-quiz-frontend-hpa
```
`<unknown>` in the `TARGETS` column for more than a couple of minutes
confirms the HPA cannot read metrics for its target.

2. **Find the root cause in the events:**
```bash
oc describe hpa <hpa-name>
```
Look for `FailedGetScale` / `unable to get metrics for resource cpu: no
matching HPA target found` referencing `scaleTargetRef`.

3. **Compare the configured target against the real Deployment name:**
```bash
oc get hpa <hpa-name> -o jsonpath='{.spec.scaleTargetRef.name}{"\n"}'
oc get deployment -l app.kubernetes.io/part-of=devops-quiz-app
```
The known failure mode: `scaleTargetRef.name` points at the **Service**
name (e.g. `devops-quiz-backend-svc`) instead of the **Deployment** name
(e.g. `devops-quiz-backend`).

4. **Patch the HPA to point at the correct Deployment** (live object only
   — this does not change the source manifest in the devops-quiz-app
   repo):
```bash
oc patch hpa <hpa-name> --type=merge \
  -p '{"spec":{"scaleTargetRef":{"name":"<correct-deployment-name>"}}}'
```

5. **Verify:**
```bash
oc get hpa devops-quiz-backend-hpa devops-quiz-frontend-hpa
```
`TARGETS` should now show a real `NN%/50%` value instead of `<unknown>`.

6. **Completion:**
Print the `oc get hpa` output in the chat, and note explicitly that this
was a live patch only — the underlying fix belongs in
`devops-quiz-app/openshift/062-HPA-backend.yml` and
`063-HPA-frontend.yml`, which should be corrected at the source so the
live patch isn't silently overwritten on the next deploy.

## Boundaries

- This is a config-level patch to a live object, not a destructive
  action, so no confirmation is required. Still: never change
  `minReplicas`, `maxReplicas`, or the metrics thresholds — only
  `scaleTargetRef.name`.
- Always flag the source-repo mismatch to the operator even after a
  successful live patch; otherwise the next CI deploy reintroduces the
  bug.

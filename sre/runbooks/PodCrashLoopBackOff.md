# Runbook: PodCrashLoopBackOff (Remediation)

This runbook applies when a pod in the managed namespace enters
`CrashLoopBackOff` or its restart count keeps climbing (any app: e.g.
devops-quiz-app, voting-app, quotes-app, web-app).

⚠️ The agent is authorized to execute steps 1-2 and the "transient failure"
branch of step 3 automatically. The other branches require operator
confirmation before acting (see boundaries below).

## Authorized Steps for the Agent:

1. **Identify the affected pod(s):**
```bash
oc get pods --field-selector=status.phase!=Running
oc get pods -l app=<app-label>
```

2. **Inspect the failure cause:**
```bash
oc logs <pod> --previous --tail=50
oc describe pod <pod>
```
Look specifically at the `Events` section and the previous container's exit
reason.

3. **Branch on the cause:**
   - **Transient failure** (e.g. the app crashed once while a dependency
     such as the database was still starting, no recurring error in the
     last 3 restarts): restart the deployment and continue to step 4.
     ```bash
     oc rollout restart deployment/<deployment-name>
     ```
   - **OOMKilled** (`Events` shows `OOMKilled`, or
     `lastState.terminated.reason == OOMKilled`): do **not** restart blindly —
     a restart will just repeat the crash. Report the pod, its current
     `resources.limits.memory`, and the measured usage to the operator and
     stop. Raising memory limits requires an operator-approved manifest
     change.
   - **ImagePullBackOff / CrashLoop from a bad image**: report the image
     reference and the exact pull error to the operator and stop. Do not
     attempt to change the image tag yourself.

4. **Verify the fix (transient-failure branch only):**
```bash
oc get pods -l app=<app-label> -w
```
Confirm the new pod reaches `Running` / `1/1 Ready` and that its restart
count does not increase again within 2 minutes of becoming ready.

5. **Completion:**
Print the final `oc get pods -l app=<app-label>` output directly in the
chat, plus a one-line summary of the root cause found in step 2.

## Boundaries

- Only the "transient failure" branch may be remediated without further
  confirmation. OOMKilled and bad-image cases are diagnosis-only — report
  and wait for the operator.
- Never edit resource limits, image tags, or any other manifest field
  live; those are source-of-truth changes that belong in the app's own
  repo.

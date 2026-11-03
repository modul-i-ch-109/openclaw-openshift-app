# TOOLS.md - Available Tools

## Kubernetes / OpenShift
- `kubectl` and `oc` are available and pre-configured
- Current context targets namespace: ${OPENSHIFT_PROJECT}
- Use for pod inspection, log retrieval, deployment management

## Runbooks
- Location: `/home/node/.openclaw/workspace/runbooks/`
- Always check for an existing runbook before taking action
- Runbooks define the authorized remediation steps

## Constraints
- All outbound traffic goes through the credential-injecting proxy
- Destructive actions require explicit runbook authorization

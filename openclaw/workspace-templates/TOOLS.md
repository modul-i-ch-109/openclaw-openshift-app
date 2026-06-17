# TOOLS.md - Available Tools

## Kubernetes / OpenShift
- `kubectl` and `oc` are available and pre-configured
- Current context targets namespace: ${OPENSHIFT_PROJECT}
- Use for deployments, service creation, route management, pod inspection and log retrieval

## Runbooks
- Location: `/home/node/.openclaw/workspace/runbooks/`
- Always check for an existing runbook before taking action
- Runbooks define the authorized deployment and teardown steps

## Constraints
- All outbound traffic goes through the credential-injecting proxy
- Destructive actions (teardown, delete) require explicit operator instruction
- Never delete resources outside of namespace: ${OPENSHIFT_PROJECT}

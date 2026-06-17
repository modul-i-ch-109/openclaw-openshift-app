# Runbook: Deploy Application on OpenShift

This runbook defines the authorized steps for deploying an application on OpenShift.

⚠️ The agent is authorized to automatically execute all steps listed below WITHOUT further confirmation.

---

## Step 1 — Namespace Check

Verify the target namespace exists and has sufficient quota:

```bash
oc get namespace ${OPENSHIFT_PROJECT}
oc describe resourcequota -n ${OPENSHIFT_PROJECT}
```

If the namespace does not exist or quota is exhausted, stop and report to the operator.

---

## Step 2 — Deploy Application

Create the Deployment using the image and configuration provided by the operator:

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <app-name>
  namespace: ${OPENSHIFT_PROJECT}
  labels:
    app: <app-name>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: <app-name>
  template:
    metadata:
      labels:
        app: <app-name>
    spec:
      containers:
      - name: <app-name>
        image: <image>
        ports:
        - containerPort: <port>
EOF
```

---

## Step 3 — Create Service

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: <app-name>
  namespace: ${OPENSHIFT_PROJECT}
  labels:
    app: <app-name>
spec:
  selector:
    app: <app-name>
  ports:
  - name: http
    port: <port>
    targetPort: <port>
EOF
```

---

## Step 4 — Create Route with TLS Edge Termination

```bash
kubectl apply -f - <<EOF
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: <app-name>
  namespace: ${OPENSHIFT_PROJECT}
  labels:
    app: <app-name>
spec:
  to:
    kind: Service
    name: <app-name>
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
EOF
```

---

## Step 5 — Verify Rollout

Wait for the deployment to complete:

```bash
kubectl rollout status deployment/<app-name> -n ${OPENSHIFT_PROJECT} --timeout=3m
```

If the rollout fails, retrieve the pod logs:

```bash
kubectl get pods -n ${OPENSHIFT_PROJECT} -l app=<app-name>
kubectl logs -n ${OPENSHIFT_PROJECT} -l app=<app-name> --tail=50
```

---

## Step 6 — Output Route URL

```bash
oc get route <app-name> -n ${OPENSHIFT_PROJECT} -o jsonpath='{.spec.host}'
```

Report the full URL to the operator:
https://<route-host>

---

## Completion

Summarize in the chat:
- ✅ Deployment name and image used
- ✅ Route URL
- ❌ Any errors encountered with details

# ZLI-Modul-109/201-openclaw-openshift-app

## Technologies

<p align="left">
  <a href="https://gitlab.com">
    <img src="https://img.shields.io/badge/GitLab_CI-FC6D26?style=for-the-badge&logo=gitlab&logoColor=white" alt="GitLab CI">
  </a>
  <a href="https://github.com/features/actions">
    <img src="https://img.shields.io/badge/GitHub_Actions-2088FF?style=for-the-badge&logo=githubactions&logoColor=white" alt="GitHub Actions">
  </a>
  <a href="https://www.redhat.com/en/technologies/cloud-computing/openshift">
    <img src="https://img.shields.io/badge/OpenShift-EE0000?style=for-the-badge&logo=redhatopenshift&logoColor=white" alt="OpenShift">
  </a>
  <a href="https://kubernetes.io">
    <img src="https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white" alt="Kubernetes">
  </a>
  <a href="https://openclaw.ai">
    <img src="https://img.shields.io/badge/OpenClaw-000000?style=for-the-badge&logo=openai&logoColor=white" alt="OpenClaw">
  </a>
</p>

---

## Overview

**openclaw-openshift-app** deploys [OpenClaw](https://openclaw.ai) on OpenShift as a
self-hosted AI agent platform. Once running, OpenClaw acts as an autonomous AI agent
with direct access to the OpenShift cluster — capable of deploying applications.

The deployment is fully portable across OpenShift clusters and namespaces — the Control
UI origin is resolved dynamically at deploy time, no hardcoded URLs. Secrets are never
stored in the repository — all sensitive values are injected at deploy time via CI/CD
variables.

---

## Repository Structure

```text
.
├── .github/
│   └── workflows/
│       ├── deploy.yml
│       └── destroy.yml
├── .gitlab-ci.yml
├── README.md
├── oc/
│   └── 01-with-deployment/
│       ├── 010-PVC.yaml
│       ├── 021-NetworkPolicy-Clawy.yaml
│       ├── 022-NetworkPolicy-ClawProxy.yaml
│       ├── 031-Deployment-Claw.yaml
│       ├── 032-Deployment-Proxy.yaml
│       ├── 041-Service-Claw.yaml
│       ├── 042-Service-Proxy.yaml
│       ├── 050-Route.yaml
│       ├── 060-HPA.yaml
│       ├── 071-Config-Map-Claw-Config.yaml     ← ws templates injected here by pipeline
│       ├── 072-Config-Map-Claw-KubeConfig.yaml
│       └── 073-Config-Map-Claw-ProxyConfig.yaml
├── openclaw/
│   └── workspace-templates/
│       ├── AGENTS.md
│       ├── IDENTITY.md
│       ├── SOUL.md
│       ├── TOOLS.md
│       └── USER.md
└── sre/
    └── runbooks/
        └── Deploy.md
```

> **Note:** No secrets are stored in the repository. All secrets (`claw-gateway-token`,
> `claw-proxy-ca`, `workspace-kubeconfig`, `llm-key`) are created by the pipeline
> from CI/CD variables.

---

## Prerequisites

### Required OpenShift Setup (one-time, per namespace)

Create the ServiceAccount and assign the required role:

```bash
oc login --server=https://api.your-cluster:6443
oc project $OPENSHIFT_PROJECT

oc create serviceaccount gitlab-pipeline-sa

oc apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-pipeline-token
  annotations:
    kubernetes.io/service-account.name: "gitlab-pipeline-sa"
type: kubernetes.io/service-account-token
EOF

oc adm policy add-role-to-user edit -z gitlab-pipeline-sa -n $OPENSHIFT_PROJECT
```

Retrieve the token for the CI/CD variable `OPENSHIFT_TOKEN`:

```bash
oc get secret gitlab-pipeline-token -o jsonpath='{.data.token}' | base64 --decode
```

### Generate Proxy CA (one-time, per cluster)

The MITM proxy requires a self-signed CA certificate pair. Generate it once and store
the values as CI/CD variables:

```bash
openssl ecparam -name prime256v1 -genkey -noout -out ca.key
openssl req -new -x509 -key ca.key -out ca.crt -days 3650 \
  -subj "/O=Claw/CN=Claw Proxy" \
  -addext "basicConstraints=critical,CA:TRUE" \
  -addext "keyUsage=critical,keyCertSign,cRLSign"

cat ca.crt  # → OPENCLAW_PROXY_CA_CRT
cat ca.key  # → OPENCLAW_PROXY_CA_KEY

rm ca.crt ca.key
```

### Generate Gateway Token (one-time)

```bash
openssl rand -hex 32  # → OPENCLAW_TOKEN
```

### CI/CD Variables

The following variables are required for both GitLab CI and GitHub Actions:

| Variable | Description | Masked |
|---|---|---|
| `OPENSHIFT_TOKEN` | Token for `gitlab-pipeline-sa` | ✅ |
| `OPENSHIFT_SERVER` | API endpoint of the cluster (`https://api.your-cluster:6443`) | ✅ |
| `OPENSHIFT_PROJECT` | Target namespace | ✅ |
| `OPENCLAW_TOKEN` | Gateway authentication token (`openssl rand -hex 32`) | ✅ |
| `OPENCLAW_PROXY_CA_CRT` | PEM content of proxy CA certificate | ❌ |
| `OPENCLAW_PROXY_CA_KEY` | PEM content of proxy CA private key | ❌ |
| `AZURE_API_KEY` | Azure OpenAI API key | ✅ |
| `AZURE_OPENAI_DEPLOYMENT` | Azure OpenAI deployment name (e.g. `gpt-4o-mini`) | ✅ |
| `AZURE_OPENAI_ENDPOINT` | Azure OpenAI endpoint hostname | ✅ |
| `AGENT_NAME` | Display name of the AI agent (e.g. `Claw`) | ❌ |
| `ORGANISATION_NAME` | Organisation name injected into agent templates | ❌ |
| `OPERATOR_NAME` | Primary operator name injected into agent templates | ❌ |

#### GitLab

Configure under **Settings → CI/CD → Variables**. Mark all variables as **Protected**.

#### GitHub Actions

Configure under **Settings → Secrets and variables → Actions → New repository secret**.

### Manual Deployment Variables

```bash
oc login --server=https://api.your-cluster:6443
export OPENSHIFT_SERVER=$(oc whoami --show-server)
export OPENSHIFT_TOKEN=$(oc whoami --show-token)
export OPENSHIFT_PROJECT=$(oc project -q)
export OPENCLAW_TOKEN="$(openssl rand -hex 32)"
export OPENCLAW_PROXY_CA_CRT="$(cat ca.crt)"
export OPENCLAW_PROXY_CA_KEY="$(cat ca.key)"
export AZURE_API_KEY="..."
export AZURE_OPENAI_DEPLOYMENT="gpt-4o-mini"
export AZURE_OPENAI_ENDPOINT="hub-ai-foundry-central.cognitiveservices.azure.com"
export AGENT_NAME="Claw"
export ORGANISATION_NAME="ZLI AIOps Demo"
export OPERATOR_NAME="Ben Schwarz"
```

---

## Deployment

### Option 1 — GitLab CI/CD Pipeline

The pipeline triggers automatically on every push to `main`.

#### Pipeline Stages & Jobs

* **`deploy`**
    * `deploy:openshift` — Creates all secrets from GitLab variables, substitutes
      variables in ConfigMaps, resolves the Route origin dynamically, injects workspace
      templates and runbooks into the ConfigMap, applies all manifests, and waits for
      rollout to complete.
* **`destroy`**
    * `destroy:openshift` — **Manual trigger.** Removes all resources labelled `app=claw`
      from the cluster.

Push to `main` to trigger a deploy. The destroy job must be started manually in the GitLab pipeline UI.

---

### Option 2 — GitHub Actions

The workflow triggers automatically on every push to `main` and can also be triggered
manually via **Actions → Deploy OpenClaw → Run workflow**.

#### Workflows

* **`deploy.yml`** — Triggered on push to `main` or manually. Creates all secrets,
  substitutes variables, injects templates, applies all manifests, and waits for rollout.
* **`destroy.yml`** — **Manual trigger only.** Go to **Actions → Destroy OpenClaw → Run workflow**
  to remove all resources labelled `app=claw` from the cluster.

---

### Option 3 — Manual Deployment (CLI)

#### 1. Login and set variables

```bash
oc login --server=https://api.your-cluster:6443
export OPENSHIFT_SERVER=$(oc whoami --show-server)
export OPENSHIFT_TOKEN=$(oc whoami --show-token)
export OPENSHIFT_PROJECT=$(oc project -q)
```

#### 2. Create secrets

```bash
# Gateway token
oc create secret generic claw-gateway-token \
  --from-literal=token=$OPENCLAW_TOKEN \
  --dry-run=client -o yaml | oc apply -f -
oc label secret claw-gateway-token app=claw app.kubernetes.io/part-of=claw --overwrite

# Proxy CA
echo "$OPENCLAW_PROXY_CA_CRT" > /tmp/ca.crt
echo "$OPENCLAW_PROXY_CA_KEY" > /tmp/ca.key
oc create secret generic claw-proxy-ca \
  --from-file=ca.crt=/tmp/ca.crt \
  --from-file=ca.key=/tmp/ca.key \
  --dry-run=client -o yaml | oc apply -f -
oc label secret claw-proxy-ca app=claw app.kubernetes.io/part-of=claw --overwrite
rm /tmp/ca.crt /tmp/ca.key

# LLM API key
oc create secret generic llm-key \
  --from-literal=custom-api-key=$AZURE_API_KEY \
  --dry-run=client -o yaml | oc apply -f -
oc label secret llm-key app=claw app.kubernetes.io/part-of=claw --overwrite

# Workspace kubeconfig
OPENSHIFT_API_HOST=$(echo $OPENSHIFT_SERVER | sed 's|https://||')
cat > /tmp/kubeconfig << EOF
apiVersion: v1
kind: Config
clusters:
- cluster:
    server: $OPENSHIFT_SERVER
    insecure-skip-tls-verify: true
  name: target
contexts:
- context:
    cluster: target
    namespace: $OPENSHIFT_PROJECT
    user: pipeline
  name: default
current-context: default
users:
- name: pipeline
  user:
    token: $OPENSHIFT_TOKEN
EOF
oc create secret generic workspace-kubeconfig \
  --from-file=kubeconfig=/tmp/kubeconfig \
  --dry-run=client -o yaml | oc apply -f -
oc label secret workspace-kubeconfig app=claw --overwrite
rm /tmp/kubeconfig
```

#### 3. Substitute variables

```bash
OPENSHIFT_API_HOST=$(echo $OPENSHIFT_SERVER | sed 's|https://||')

sed -i \
  -e "s|\${OPENSHIFT_SERVER}|$OPENSHIFT_SERVER|g" \
  -e "s|\${OPENSHIFT_PROJECT}|$OPENSHIFT_PROJECT|g" \
  -e "s|\${OPENSHIFT_API_HOST}|$OPENSHIFT_API_HOST|g" \
  oc/01-with-deployment/072-Config-Map-Claw-KubeConfig.yaml \
  oc/01-with-deployment/071-Config-Map-Claw-Config.yaml \
  oc/01-with-deployment/073-Config-Map-Claw-ProxyConfig.yaml

for f in openclaw/workspace-templates/*.md; do
  sed -i \
    -e "s|\${OPENSHIFT_PROJECT}|$OPENSHIFT_PROJECT|g" \
    -e "s|\${AGENT_NAME}|$AGENT_NAME|g" \
    -e "s|\${ORGANISATION_NAME}|$ORGANISATION_NAME|g" \
    -e "s|\${OPERATOR_NAME}|$OPERATOR_NAME|g" \
    -e "s|\${AZURE_OPENAI_DEPLOYMENT}|$AZURE_OPENAI_DEPLOYMENT|g" \
    -e "s|\${AZURE_OPENAI_ENDPOINT}|$AZURE_OPENAI_ENDPOINT|g" \
    "$f"
done
```

#### 4. Resolve the allowed origin

```bash
ROUTE_HOST=$(oc get route claw -n $OPENSHIFT_PROJECT \
  -o jsonpath='{.spec.host}' 2>/dev/null || echo "")

if [ -z "$ROUTE_HOST" ]; then
  ALLOWED_ORIGIN="ALLOWED_ORIGIN_PLACEHOLDER"
else
  ALLOWED_ORIGIN="https://$ROUTE_HOST"
fi

sed -i "s|ALLOWED_ORIGIN_PLACEHOLDER|$ALLOWED_ORIGIN|g" \
  oc/01-with-deployment/071-Config-Map-Claw-Config.yaml
```

#### 5. Inject workspace templates into ConfigMap

```bash
for f in openclaw/workspace-templates/*.md; do
  filename=$(basename "$f")
  echo "  _ws_${filename}: |" >> oc/01-with-deployment/071-Config-Map-Claw-Config.yaml
  sed 's/^/    /' "$f" >> oc/01-with-deployment/071-Config-Map-Claw-Config.yaml
done

for f in sre/runbooks/*.md; do
  [ -f "$f" ] || continue
  filename=$(basename "$f")
  echo "  _ws_runbooks--${filename}: |" >> oc/01-with-deployment/071-Config-Map-Claw-Config.yaml
  sed 's/^/    /' "$f" >> oc/01-with-deployment/071-Config-Map-Claw-Config.yaml
done
```

#### 6. Apply all resources

```bash
oc apply -f oc/01-with-deployment/
```

If this is a first-time deploy (Route did not exist yet), resolve the origin after
the Route has been created and re-apply the ConfigMap:

```bash
ROUTE_HOST=$(oc get route claw -n $OPENSHIFT_PROJECT -o jsonpath='{.spec.host}')
sed -i "s|ALLOWED_ORIGIN_PLACEHOLDER|https://$ROUTE_HOST|g" \
  oc/01-with-deployment/071-Config-Map-Claw-Config.yaml
oc apply -f oc/01-with-deployment/071-Config-Map-Claw-Config.yaml
oc rollout restart deployment/claw
```

#### 7. Retrieve the gateway token

```bash
oc get secret claw-gateway-token -o jsonpath='{.data.token}' | base64 --decode
```

Open the Route URL in your browser and enter the token when prompted.

---

## Teardown

### Via GitLab Pipeline

Trigger the `destroy:openshift` job manually in the GitLab pipeline UI.

### Via GitHub Actions

Go to **Actions → Destroy OpenClaw → Run workflow**.

### Via CLI

```bash
oc delete deployment,service,route,secret,configmap,networkpolicy,pvc \
  -l app=claw --ignore-not-found
```

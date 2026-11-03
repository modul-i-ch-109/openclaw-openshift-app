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
├── openshift/
│   ├── 010-PVC.yml
│   ├── 021-NetworkPolicy-Claw.yml
│   ├── 022-NetworkPolicy-ClawProxy.yml
│   ├── 031-Deployment-Claw.yml
│   ├── 032-Deployment-Proxy.yml
│   ├── 041-Service-Claw.yml
│   ├── 042-Service-Proxy.yml
│   ├── 050-Route.yml
│   ├── 060-HPA.yml
│   ├── 071-Config-Map-Claw-Config.yml     ← ws templates + runbooks injected here by pipeline
│   ├── 072-Config-Map-Claw-KubeConfig.yml
│   └── 073-Config-Map-Claw-ProxyConfig.yml
├── openclaw/
│   └── workspace-templates/
│       ├── AGENTS.md
│       ├── IDENTITY.md
│       ├── SOUL.md
│       ├── TOOLS.md
│       └── USER.md
└── sre/
    └── runbooks/
        ├── HPAInvalidScaleTarget.md
        ├── PodCrashLoopBackOff.md
        └── PostgresHAClusterDegraded.md
```

> **Note:** No secrets are stored in the repository. All secrets (`claw-gateway-token`,
> `claw-proxy-ca`, `workspace-kubeconfig`, `llm-key`) are created by the pipeline
> from CI/CD variables.

---

## Prerequisites

### Model Provider Setup (one-time per credential)

OpenClaw is configured to use **Claude, authenticated against your
existing claude.ai Pro/Max subscription** rather than a pay-per-token
API key, via OpenClaw's own built-in `anthropic` provider directly (no
custom `models.providers` config, no proxy credential injection
needed).

> Earlier versions of this repo used a single hardcoded Azure OpenAI
> deployment behind a custom MITM-proxy-injected credential. That's gone
> - found live, this subscription's Azure tenant enforces a hard,
> account-wide ~50K-tokens/minute rate limit that made Azure OpenAI
> unusable here regardless of model/deployment/SKU. A plain OpenAI API
> key was tried next, but that's usage-billed per token - found live,
> that silently burns through account credits and then hard-fails with
> "no credits remaining" once they're gone. Anthropic's subscription
> setup-token avoids that: usage comes out of the existing Claude
> Pro/Max allowance, not separate API billing.

#### Claude (your claude.ai Pro/Max subscription, no separate API key)

OpenClaw supports authenticating Anthropic's `anthropic` provider with a
**subscription setup-token** instead of a pay-per-token API key - this
uses your existing Claude Pro/Max subscription's usage allowance, not
separate API billing. Generate it **on your own machine** (the Claude
Code CLI needs to be installed there and logged into your subscription,
NOT inside the cluster):

```bash
npm install -g @anthropic-ai/claude-code   # if not already installed
claude login                                # log into your claude.ai subscription, if not already
claude setup-token                          # prints a token to stdout
```

Set the printed token as the `ANTHROPIC_SETUP_TOKEN` CI/CD variable. It
gets registered into OpenClaw's own `auth-profiles.json` by an init
container (`init-model-auth` in `031-Deployment-Claw.yml`) on every pod
start - re-run `claude setup-token` and update the variable if it ever
expires (`openclaw models status --check` from inside the pod reports
expiry).

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
| `ANTHROPIC_SETUP_TOKEN` | Claude subscription setup-token (`claude setup-token`) | ✅ |
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
export ANTHROPIC_SETUP_TOKEN="..."          # from `claude setup-token`, see "Model Provider Setup" above
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

# LLM credential
oc create secret generic llm-key \
  --from-literal=anthropic-setup-token=$ANTHROPIC_SETUP_TOKEN \
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
  openshift/072-Config-Map-Claw-KubeConfig.yml \
  openshift/071-Config-Map-Claw-Config.yml \
  openshift/073-Config-Map-Claw-ProxyConfig.yml

for f in openclaw/workspace-templates/*.md; do
  sed -i \
    -e "s|\${OPENSHIFT_PROJECT}|$OPENSHIFT_PROJECT|g" \
    -e "s|\${AGENT_NAME}|$AGENT_NAME|g" \
    -e "s|\${ORGANISATION_NAME}|$ORGANISATION_NAME|g" \
    -e "s|\${OPERATOR_NAME}|$OPERATOR_NAME|g" \
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
  openshift/071-Config-Map-Claw-Config.yml
```

#### 5. Inject workspace templates into ConfigMap

```bash
for f in openclaw/workspace-templates/*.md; do
  filename=$(basename "$f")
  echo "  _ws_${filename}: |" >> openshift/071-Config-Map-Claw-Config.yml
  sed 's/^/    /' "$f" >> openshift/071-Config-Map-Claw-Config.yml
done

for f in sre/runbooks/*.md; do
  [ -f "$f" ] || continue
  filename=$(basename "$f")
  echo "  _ws_runbooks--${filename}: |" >> openshift/071-Config-Map-Claw-Config.yml
  sed 's/^/    /' "$f" >> openshift/071-Config-Map-Claw-Config.yml
done
```

#### 6. Apply all resources

```bash
oc apply -f openshift/
```

If this is a first-time deploy (Route did not exist yet), resolve the origin after
the Route has been created and re-apply the ConfigMap:

```bash
ROUTE_HOST=$(oc get route claw -n $OPENSHIFT_PROJECT -o jsonpath='{.spec.host}')
sed -i "s|ALLOWED_ORIGIN_PLACEHOLDER|https://$ROUTE_HOST|g" \
  openshift/071-Config-Map-Claw-Config.yml
oc apply -f openshift/071-Config-Map-Claw-Config.yml
oc rollout restart deployment/claw
```

#### 7. Retrieve the gateway token

```bash
oc get secret claw-gateway-token -o jsonpath='{.data.token}' | base64 --decode
```

Open the Route URL in your browser and enter the token when prompted.

#### 8. Approve your browser (one-time per fresh state)

Even with a valid token, OpenClaw requires a one-time **device pairing**
approval for every browser the Gateway hasn't seen before — this is a
deliberate security boundary (`gateway.auth.mode: token` does not support
auto-approving browser/Control UI pairing; only a trusted
identity-aware reverse proxy setup can, which this deployment does not
use). You'll see a "Diesen Browser genehmigen" / "Approve this browser"
screen after entering the token. It happens:

- the first time you ever open the Control UI, and
- again after any teardown that deletes the `claw-home-pvc` (device
  pairings are stored there, so a fresh PVC means a fresh, empty device
  table).

Approve it from your own terminal (no need to ask anyone else — if you
have `oc exec` access to the namespace, you already have full admin
over the Gateway). Make sure the pod is `1/1 Running` first
(`oc get pods -l app=claw`) — right after a deploy/rollout it can still
be starting up, and `oc exec` fails with "container not found" until
the gateway container is actually up:

```bash
oc exec deployment/claw -c gateway -- node /app/dist/index.js devices approve --latest
```

Or list pending requests first if you want to confirm the remote IP
before approving a specific one (run these one at a time, not pasted
together — the second line needs the first one's output):

```bash
oc exec deployment/claw -c gateway -- node /app/dist/index.js devices list --json
```

```bash
oc exec deployment/claw -c gateway -- node /app/dist/index.js devices approve <requestId>
```

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

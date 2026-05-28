---
name: setup-openfaas
description: "Installs and configures OpenFaaS into an existing Kubernetes cluster via Helm. Supports OpenFaaS Community Edition (CE) and OpenFaaS Standard / For Enterprises (Pro), including core components: gateway, operator, autoscaler, JetStream queue-worker, dashboard, and NATS. Use when asked to install, deploy, set up, or configure an OpenFaaS cluster."
---

# Setup OpenFaaS

Installs and configures OpenFaaS into an existing Kubernetes cluster using the official [openfaas Helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas).

This skill covers:

- **OpenFaaS Community Edition (CE)** — single-replica install for exploration and development.
- **OpenFaaS Pro** — production-grade install with the Pro gateway, operator + Function CRD, Pro autoscaler, Pro JetStream queue-worker, and the Pro dashboard.

OpenFaaS Pro ships in two licensed variants — **Standard** (single-team, single-tenant) and **For Enterprises** (multi-tenant, regulated, SLA-backed). Both are deployed through **the same Helm chart with the same `openfaasPro: true` flag**; the difference is in the license file itself (which unlocks features at runtime) and a small set of optional values. See [Standard vs For Enterprises](#standard-vs-for-enterprises) below for the specific values that matter at install time.

The following are out of scope for this skill and are installed from separate Helm charts:

- Event connectors (Kafka, SQS, SNS, PostgreSQL, RabbitMQ, Pub/Sub, Cron, ...)
- Dedicated JetStream queue-workers
- Pro Function Builder API
- External NATS cluster
- Ingress, TLS, and DNS setup
- IAM **Policy** and **Role** custom resources, and CI/CD Web Identity Federation (GitHub Actions, GitLab CI). The skill enables and configures IAM in the chart, but creating the per-user/per-team Policies and Roles is a post-install task.

## Prerequisites

- An existing Kubernetes cluster with a working kubeconfig.
- `helm` (v3) and `kubectl` on PATH.
- For Pro installs: an OpenFaaS Pro license file (typically `~/.openfaas/LICENSE`).
- `faas-cli` on PATH if you want to log in to the gateway at the end.

## Parameters the User May Specify

Before starting, confirm with the user (or accept defaults):

- **Edition** — `ce` (Community Edition) or `pro` (Standard / For Enterprises). Default: `ce`. If the user specifies `standard` or `enterprise(s)`, treat both as `pro` and apply the extra values noted in [Standard vs For Enterprises](#standard-vs-for-enterprises).
- **Kubeconfig path** — path to the kubeconfig for the target cluster. If not provided, use the current `KUBECONFIG` / default kubeconfig.
- **Values directory** — directory where the generated Helm values file should be stored. Default: current working directory.
- **License file** (Pro only) — path to the OpenFaaS Pro license. Default: `~/.openfaas/LICENSE`.
- **Dashboard** (Pro only) — enable the OpenFaaS Pro dashboard? Default: `true`.
- **IAM / SSO** (Pro only, For Enterprises license required) — enable Identity and Access Management? Default: `false`. If enabled, also collect:
  - **Gateway public URL** — used as `iam.systemIssuer.url`, e.g. `https://gateway.openfaas.example.com`.
  - **Dashboard OIDC issuer URL** — your IdP's issuer, e.g. `https://example.eu.auth0.com/` or `https://keycloak.example.com/realms/openfaas`.
  - **Dashboard OAuth client ID**.
  - **Dashboard OAuth client secret** (only if the IdP requires one).
  - **Dashboard OIDC scopes** — defaults to `[openid, profile, email]`.
  - **Dashboard public URL** — required when IAM is enabled, e.g. `https://dashboard.openfaas.example.com`.
- **Helm values overrides** — any additional values from the chart's [values.yaml](https://github.com/openfaas/faas-netes/blob/master/chart/openfaas/values.yaml) the user wants to customize.

> **Prompt for missing values.** If any required value above cannot be derived from the user's request, the environment, or sensible defaults, ask the user for it before proceeding. Don't guess at things like the gateway's public URL, OIDC client IDs/secrets, IdP issuer URLs, license paths, or kubeconfig locations — these are install-time choices the user must make. Group related questions into a single prompt where possible.

## Workflow

The workflow branches on **edition**. Steps 1, 2 and the final verify/login steps are shared.

### Step 1: Verify cluster access

If the user supplied a kubeconfig path, export it:

```bash
export KUBECONFIG=<path-to-kubeconfig>
```

Verify cluster access:

```bash
kubectl get nodes
```

### Step 2: Create OpenFaaS namespaces

```bash
kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml
```

This creates the `openfaas` (core services) and `openfaas-fn` (functions) namespaces.

### Step 3: Add the Helm repository

```bash
helm repo add openfaas https://openfaas.github.io/faas-netes/
helm repo update
```

### Step 4 (edition-specific): Configure and deploy

Pick **4A** for Community Edition, or **4B** for Standard / For Enterprises.

---

#### 4A. Deploy OpenFaaS Community Edition (CE)

CE has sensible defaults baked into the chart and does not require a license or a values overlay. If the user requested overrides, create a small `<values-dir>/openfaas-values.yaml` containing only those overrides; otherwise skip the file.

Deploy:

```bash
helm upgrade --install openfaas openfaas/openfaas \
  --namespace openfaas
```

If you have an overrides file:

```bash
helm upgrade --install openfaas openfaas/openfaas \
  --namespace openfaas \
  -f <values-dir>/openfaas-values.yaml
```

Skip ahead to [Step 5: Verify the deployment](#step-5-verify-the-deployment).

---

#### 4B. Deploy OpenFaaS Standard / For Enterprises (Pro)

**4B.1 — Create the license secret**

```bash
kubectl create secret generic -n openfaas openfaas-license \
  --from-file license="$HOME/.openfaas/LICENSE"
```

**4B.2 — (Dashboard only) Create the dashboard signing key**

Skip this if the user does not want the dashboard, or accepts dev-mode auto-generated keys (sessions invalidate on dashboard restart).

```bash
# Generate a private key and corresponding public key
openssl ecparam -genkey -name prime256v1 -noout -out jwt_key
openssl ec -in jwt_key -pubout -out jwt_key.pub

# Store both in a secret in the openfaas namespace
kubectl -n openfaas create secret generic dashboard-jwt \
  --from-file=key=./jwt_key \
  --from-file=key.pub=./jwt_key.pub

# Remove local key material once stored in the cluster
rm jwt_key jwt_key.pub
```

**4B.3 — Create the values file**

Create `<values-dir>/openfaas-values.yaml` with the recommended Pro defaults. Apply any user-requested overrides on top.

```yaml
openfaasPro: true
clusterRole: true

operator:
  create: true
  leaderElection:
    enabled: true

gateway:
  replicas: 3
  # 10 minute timeout — adjust if your functions are shorter/longer running
  upstreamTimeout: 10m
  writeTimeout: 10m2s
  readTimeout: 10m2s

autoscaler:
  enabled: true

dashboard:
  enabled: true
  # Set to a fully qualified domain name when exposing publicly, or "localhost"
  # for port-forward-only access.
  publicURL: "localhost"
  # Omit or leave blank to auto-generate signing keys (dev only — sessions
  # invalidate on restart). Set to "dashboard-jwt" if you ran step 4B.2.
  signingKeySecret: "dashboard-jwt"

queueWorker:
  replicas: 3

queueWorkerPro:
  maxInflight: 50

# JetStream is the supported queue mode for Pro.
queueMode: jetstream

nats:
  # Must stay at 1 when using the bundled NATS — the chart deploys a single
  # NATS replica, so any higher value will leave the JetStream stream stuck
  # without enough peers. This applies even on multi-node clusters running
  # multiple replicas of other OpenFaaS components.
  # For >1 stream replication, install an external NATS cluster (out of scope).
  streamReplication: 1

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 10001
```

Notes when adjusting:

- `operator.create: true` is recommended for Pro. It enables the Function CRD; on upgrade from CE it migrates existing functions automatically.
- `operator.leaderElection.enabled: true` is **required** whenever `gateway.replicas > 1` (the operator runs in the gateway pod).
- `clusterRole: true` is required for CPU/RAM metrics (autoscaling) and for managing functions across multiple namespaces.
- `dashboard.publicURL` must be an FQDN or `localhost`. Use `localhost` for port-forward access.
- `nats.streamReplication` **must stay at `1`** when using the bundled NATS (the chart only deploys one NATS replica). Raising it — even on multi-node clusters with 3x gateway / queue-worker replicas — will break JetStream because there are not enough NATS peers. To get replication factor `3` for production resilience, install an external NATS cluster and point OpenFaaS at it (out of scope).
- Don't copy the chart's full `values.yaml` into your overrides file — it is large and the chart maintainers update it (e.g. image versions). Keep your file minimal.

Other common values to consider:

- `gateway.scaleFromZero` — enable scale-from-zero on the gateway
- `functions.imagePullPolicy: IfNotPresent` — faster cold starts
- `functions.readinessProbe.periodSeconds` / `livenessProbe.periodSeconds` — tune cold starts (see chart docs)
- `prometheus.create`, `prometheus.pvc.enabled` — control the bundled Prometheus and its persistence
- `generateBasicAuth: false` — disable the auto-generated basic-auth secret (e.g. when using GitOps and pre-creating credentials)
- `istio.mtls: true` — only when running under Istio with mTLS

**4B.4 — (Optional, For Enterprises only) Configure IAM / SSO**

Skip this step unless the user enabled the **IAM / SSO** parameter. IAM requires an **OpenFaaS for Enterprises** license and an OIDC-compatible Identity Provider (Auth0, Keycloak, Okta, Google, Microsoft Entra, etc.).

A full walkthrough — including provider setup, Policies, Roles, and CI federation — is here:
👉 <https://www.openfaas.com/blog/walkthrough-iam-for-openfaas/>

This step covers what the chart needs at install time. Before continuing, make sure the user has:

1. Created an OIDC client (application) in their IdP for OpenFaaS.
2. Allowed the following callback URLs in the IdP client:
   - `http://127.0.0.1:31111/oauth/callback` (used by `faas-cli pro auth`)
   - `https://<dashboard public URL>/auth/callback` (if the dashboard is enabled)
3. Provided the values listed under [Parameters](#parameters-the-user-may-specify) (gateway public URL, dashboard public URL, OIDC issuer, client ID, optional client secret, scopes). If any are missing, prompt the user.

**4B.4a — Create the OpenFaaS IAM signing key**

The IAM issuer needs a signing key to sign OpenFaaS API and Function Invocation JWTs.

```bash
openssl ecparam -genkey -name prime256v1 -noout -out issuer.key

kubectl -n openfaas create secret generic issuer-key \
  --from-file=issuer.key=./issuer.key

rm issuer.key
```

**4B.4b — (Dashboard only) Create the AES encryption key**

The dashboard encrypts the OpenFaaS access token in the session cookie with an AES key.

```bash
openssl rand -hex 16 > aes_key

kubectl -n openfaas create secret generic aes-key \
  --from-file=aes_key=./aes_key

rm aes_key
```

**4B.4c — (Dashboard only) Store the OIDC client secret**

Skip if the IdP does not require a client secret. Save the secret from the IdP into a file named `client_secret` (no trailing newline), then:

```bash
kubectl -n openfaas create secret generic <provider>-client-secret \
  --from-file=client_secret=./client_secret

rm client_secret
```

Use a descriptive secret name such as `keycloak-client-secret` or `auth0-client-secret` — you'll reference it by name in the values file.

**4B.4d — Add the IAM block to the values file**

Append to `<values-dir>/openfaas-values.yaml`:

```yaml
iam:
  enabled: true
  systemIssuer:
    # The gateway's public URL. Used as the issuer for OpenFaaS-issued tokens.
    url: https://gateway.openfaas.example.com

  # Required only when the dashboard is enabled and you want SSO login.
  dashboardIssuer:
    # The OIDC issuer URL of your IdP.
    url: https://keycloak.example.com/realms/openfaas
    clientId: openfaas
    # Name of the Kubernetes secret created in 4B.4c. Leave blank if the IdP
    # does not need a client secret.
    clientSecret: keycloak-client-secret
    scopes:
      - openid
      - profile
      - email
```

Also update the `dashboard` block (added in 4B.3) so the dashboard knows its public URL — IAM-protected dashboards must be reachable at an FQDN:

```yaml
dashboard:
  enabled: true
  publicURL: https://dashboard.openfaas.example.com
  signingKeySecret: dashboard-jwt
```

> If the gateway or IdP uses an internal CA / self-signed TLS certificate, the dashboard will need a `ca-bundle` secret and `caBundleSecretName` set in values. See [Custom CA bundle for OpenFaaS IAM](https://docs.openfaas.com/openfaas-pro/iam/overview/) and the linked walkthrough.

**4B.4e — Post-install (manual)**

After the chart is deployed, the user (or a follow-up task) must create:

- A **JwtIssuer** Custom Resource for each trusted IdP (the chart does not create this automatically for user IdPs).
- One or more **Policy** Custom Resources defining permissions.
- One or more **Role** Custom Resources binding policies to identities matched on JWT claims.

See the [walkthrough](https://www.openfaas.com/blog/walkthrough-iam-for-openfaas/) for concrete examples. Without at least one matching Role, users will not be able to authenticate via `faas-cli pro auth` or the dashboard.

**4B.5 — Deploy**

```bash
helm upgrade --install openfaas openfaas/openfaas \
  --namespace openfaas \
  -f <values-dir>/openfaas-values.yaml
```

#### Standard vs For Enterprises

Both variants install through the same chart with the same `openfaasPro: true` flag. The features unlocked at runtime (multi-namespace, IAM/SSO, dedicated queues, advanced security profiles, …) are gated by the contents of the **license file** itself, not by the chart values. A few install-time considerations:

- **OpenFaaS Standard**: the recommended values above are sufficient. Standard supports a single namespace for functions (`openfaas-fn`).
- **OpenFaaS for Enterprises**:
  - **Multi-namespace** support is the headline difference. It is already enabled by `clusterRole: true` in the recommended values; no extra flag is needed. To use it, label additional function namespaces with `openfaas=1`:
    ```bash
    kubectl create namespace <ns>
    kubectl label namespace <ns> openfaas=1
    ```
  - **Operator tuning for many functions**: when running large numbers of functions, raise the operator's Kubernetes-client rate limits in your values file:
    ```yaml
    operator:
      kubeClientQPS: 100    # default 100, raise as needed
      kubeClientBurst: 250  # default 250, raise as needed
    ```
  - **IAM / SSO** is an Enterprise-tier feature and is covered by [Step 4B.4](#step-4-edition-specific-configure-and-deploy) above. Creation of IAM `Policy` and `Role` Custom Resources is a follow-up task — see the [walkthrough](https://www.openfaas.com/blog/walkthrough-iam-for-openfaas/).
  - **Function Builder API**, **dedicated JetStream queue-workers**, and **air-gap / image mirroring** are also Enterprise-tier features. They are installed from separate Helm charts and are **out of scope for this skill** — install the core platform first, then layer them in afterwards.

If the user is unsure which variant their license is for, the install steps are identical — just proceed with the Pro path. Features that the license does not entitle will simply not activate.

---

### Step 5: Verify the deployment

```bash
kubectl get pods -n openfaas
```

Wait for all pods to be Running/Ready. Some pods (dashboard, queue-worker) may show `CrashLoopBackOff` during the first 30–60 seconds while their dependencies (NATS, gateway) start up. This is expected — wait and re-check before reporting a problem.

Optionally run the [OpenFaaS config checker](https://github.com/openfaas/config-checker) to sanity-check the installation.

### Step 6: Log in with faas-cli

With the default `NodePort` service type the gateway is reachable on port `31112` on any node:

```bash
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
export OPENFAAS_URL=http://${NODE_IP}:31112

PASSWORD=$(kubectl get secret -n openfaas basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode)
echo -n "$PASSWORD" | faas-cli login --username admin --password-stdin
```

If `NodePort` is not reachable (e.g. nodes are not directly addressable), port-forward instead:

```bash
kubectl port-forward -n openfaas svc/gateway 8080:8080
export OPENFAAS_URL=http://127.0.0.1:8080
```

### Step 7: (Pro + dashboard) Access the dashboard

If the dashboard was enabled with `publicURL: "localhost"`:

```bash
kubectl port-forward -n openfaas svc/dashboard 8081:8080
```

Open `http://127.0.0.1:8081` and log in with username `admin` and the same password used for `faas-cli login`.

### Step 8: Report back to the user

Report:

- Edition installed (CE or Pro) and any non-default values applied
- Values file path (if one was created)
- Gateway URL (e.g. `http://<node-ip>:31112` or the port-forward URL)
- Dashboard URL (Pro only, if enabled)
- The exact `helm upgrade --install` command used, so the user can re-apply values later

## Upgrading or changing values

To upgrade the chart or change values on an existing deployment, edit the values file and re-run the same command:

```bash
export KUBECONFIG=<path-to-kubeconfig>
helm repo update
helm upgrade --install openfaas openfaas/openfaas \
  --namespace openfaas \
  -f <values-dir>/openfaas-values.yaml   # omit -f for a plain CE install
```

The OpenFaaS team recommends upgrading every 1–2 weeks for fixes and features. There is no need to uninstall before upgrading.

The one exception: switching from `clusterRole: false` to `clusterRole: true` may require deleting conflicting RBAC objects one by one as `helm` reports them, then re-running the upgrade.

## Important Notes

- For Pro: the license file is required. The chart will not deploy Pro components without the `openfaas-license` secret.
- The default `serviceType` is `NodePort` on port `31112`.
- The chart's pre-install hook auto-generates the `basic-auth` secret unless `generateBasicAuth: false` is set. For GitOps installs, pre-create the secret so credentials remain stable.
- The bundled NATS runs as a single replica and is suitable for dev/staging and light production. Because there is only one NATS peer, `nats.streamReplication` must stay at `1` regardless of cluster size or how many replicas you run of the other OpenFaaS components. For critical production workloads needing JetStream replication, install a dedicated multi-replica NATS cluster (out of scope for this skill).
- The non-operator "controller" mode used in CE is deprecated in Pro. Use `operator.create: true` for new Pro installs.

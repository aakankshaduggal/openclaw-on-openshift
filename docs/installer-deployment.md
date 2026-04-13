# Deploying OpenClaw on OpenShift with openclaw-installer

Step-by-step guide for deploying OpenClaw on an OpenShift cluster using the [openclaw-installer](https://github.com/aakankshaduggal/openclaw-installer).

## Prerequisites

- OpenShift cluster (4.17+) — namespace-scoped access is sufficient, no cluster-admin needed
- `oc` CLI authenticated (`oc login`)
- A model serving endpoint (vLLM, KServe, or external API key)
- Node.js 22+ on your local machine (to run the installer)

## Step 1: Create your namespace

```bash
oc login --token=<your-token> --server=https://api.<your-cluster>:443
oc new-project <your-name>-openclaw
```

## Step 2: Start the installer

```bash
git clone https://github.com/aakankshaduggal/openclaw-installer.git
cd openclaw-installer
npm install && npm run build && npm run dev
```

Open `http://localhost:3000`. The installer auto-detects your OpenShift cluster and loads the OpenShift provider plugin.

## Step 3: Fill in the deploy form

| Field | Value | Notes |
|-------|-------|-------|
| **Agent name** | your choice | e.g., `my-agent` |
| **Project / Namespace** | your namespace | e.g., `aduggal-openclaw` |
| **Image** | `ghcr.io/openclaw/openclaw:latest` | |
| **Provider** | Self-hosted (vLLM) | or Anthropic/OpenAI if using cloud |
| **Model endpoint** | your vLLM URL | e.g., `https://vllm-20b-gpt-oss.apps.rosa.<cluster>/v1` |

Click **Deploy**. The installer streams logs as it creates each resource.

## Step 4: Fix the Route (if needed)

If you see "Application is not available" when opening the Route URL, the TLS termination may be misconfigured. See [troubleshooting.md](troubleshooting.md#route-returns-application-is-not-available-503).

## Step 5: Approve device pairing

After SSO login, the Control UI requires a one-time device pairing. Get the request ID from the UI prompt, then:

```bash
oc exec deployment/openclaw -n <namespace> -c gateway -- \
  openclaw devices approve <request-id>
```

Or use the **Open** action from the installer's **Instances** tab.

## Step 6: Verify model configuration

Check the gateway logs to confirm the correct model is loaded:

```bash
oc logs deployment/openclaw -c gateway -n <namespace> | grep "agent model"
# Expected: agent model: openai-compat/gpt-oss-20b
```

If it shows a different model (e.g., `anthropic/claude-sonnet-4-6`), see [troubleshooting.md](troubleshooting.md#gateway-uses-wrong-model--no-api-key-errors).

## What the installer creates

The OpenShift deployer creates these resources:

| Resource | Purpose |
|----------|---------|
| `ServiceAccount/openclaw-oauth-proxy` | SA for OAuth with redirect annotation |
| `Secret/openclaw-oauth-config` | OAuth client-secret + cookie secret |
| `Secret/openclaw-secrets` | Gateway token + provider API keys |
| `ConfigMap/openclaw-config` | `openclaw.json` gateway configuration |
| `ConfigMap/openclaw-agent` | Agent workspace files |
| `PVC/openclaw-home-pvc` | 10Gi persistent state |
| `Service/openclaw` | ClusterIP: gateway (18789) + oauth-ui (8443) |
| `Route/openclaw` | TLS-terminated route to oauth-proxy |
| `Deployment/openclaw` | Init container + oauth-proxy sidecar + gateway |

## Instance management

From the installer's **Instances** tab:

| Action | Effect |
|--------|--------|
| **Open** | Opens Route URL with token pre-filled |
| **Re-deploy** | Syncs local agent workspace files + restarts pod |
| **Stop** | Scales replicas to 0 |
| **Start** | Scales replicas back to 1 |
| **Approve Pairing** | Approves pending device pairing requests |

# Vertex AI Provider for OpenClaw on OpenShift

Use Google Vertex AI (Gemini models) as the model provider for OpenClaw on OpenShift. This avoids self-hosting models entirely — compute happens on Google's infrastructure.

## Prerequisites

- GCP project with Vertex AI API enabled
- GCP service account with `roles/aiplatform.user` (or equivalent)
- Service account JSON key file
- OpenShift cluster with outbound HTTPS to `*.googleapis.com`

## Architecture

```
┌──────────────────────────────────┐
│         OpenShift Pod            │
│                                  │
│  ┌─────────────┐                │
│  │   gateway    │                │
│  │   port 18789 │                │
│  └──────┬──────┘                │
│         │                        │
│  ┌──────▼──────┐                │
│  │  GCP SA key  │                │
│  │  (Secret)    │                │
│  └──────┬──────┘                │
└─────────┼────────────────────────┘
          │ HTTPS
          ▼
┌──────────────────────────────────┐
│  Vertex AI (Google Cloud)        │
│  gemini-2.5-pro / gemini-2.5-flash│
└──────────────────────────────────┘
```

No GPU nodes needed on the cluster. No model downloads. No vLLM.

> **Status:** Experimental. OpenClaw's `google-generative-ai` API adapter does not natively support GCP service account authentication for Vertex AI. The supported path for Gemini models is **Google AI Studio** with a `GEMINI_API_KEY`. See [Known Limitation](#known-limitation-vertex-ai-auth) below.

## Setup Steps

### Step 1: Create a GCP service account (if needed)

```bash
# Create the SA
gcloud iam service-accounts create openclaw-vertex \
  --display-name="OpenClaw Vertex AI" \
  --project=YOUR_PROJECT_ID

# Grant Vertex AI access
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:openclaw-vertex@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/aiplatform.user"

# Create a key file
gcloud iam service-accounts keys create /tmp/gcp-sa.json \
  --iam-account=openclaw-vertex@YOUR_PROJECT_ID.iam.gserviceaccount.com
```

### Step 2: Create the secret on OpenShift

```bash
oc create secret generic gcp-sa \
  --from-file=sa.json=/tmp/gcp-sa.json \
  -n <namespace>
```

Clean up the local key after creating the secret:
```bash
rm /tmp/gcp-sa.json
```

### Step 3: Update the ConfigMap

Patch `openclaw-config` to use the Vertex AI provider:

```json
{
  "agents": {
    "defaults": {
      "model": "google-vertex/gemini-2.5-pro"
    }
  },
  "models": {
    "mode": "replace",
    "providers": {
      "google-vertex": {
        "baseUrl": "https://us-east1-aiplatform.googleapis.com/v1/projects/YOUR_PROJECT_ID/locations/us-east1/publishers/google/models",
        "api": "google-vertex",
        "auth": "gcp-service-account",
        "serviceAccountKeyFile": "/secrets/gcp/sa.json",
        "models": [
          {
            "id": "gemini-2.5-pro",
            "name": "gemini-2.5-pro",
            "reasoning": false,
            "input": ["text", "image"],
            "contextWindow": 1048576,
            "maxTokens": 8192
          },
          {
            "id": "gemini-2.5-flash",
            "name": "gemini-2.5-flash",
            "reasoning": false,
            "input": ["text", "image"],
            "contextWindow": 1048576,
            "maxTokens": 8192
          }
        ]
      }
    }
  }
}
```

### Step 4: Mount the secret in the Deployment

Add the GCP SA secret as a volume mount in the gateway container:

```yaml
# In the Deployment spec
containers:
  - name: gateway
    volumeMounts:
      - name: gcp-sa
        mountPath: /secrets/gcp
        readOnly: true
volumes:
  - name: gcp-sa
    secret:
      secretName: gcp-sa
```

### Step 5: Restart the deployment

```bash
oc rollout restart deployment/openclaw -n <namespace>
```

Verify:
```bash
oc logs deployment/openclaw -c gateway -n <namespace> | grep "agent model"
# Should show: agent model: google-vertex/gemini-2.5-pro
```

## Using the openclaw-installer

The installer has built-in Vertex AI support:

1. Select **Vertex AI (Gemini)** as the provider
2. Upload your GCP service account JSON file (or provide an absolute path)
3. The installer auto-extracts the `project_id` and configures everything

This is the simplest path — no manual ConfigMap or Deployment patching needed.

## Vertex AI vs. Other Providers

| | Vertex AI (Gemini) | vLLM (self-hosted) | Codex Harness |
|-|--------------------|--------------------|---------------|
| **GPU nodes needed** | No | Yes | No |
| **Cost model** | Pay-per-token | GPU node hours | Pay-per-token |
| **Latency** | ~1-3s (network) | ~0.5-1s (cluster-local) | ~1-3s (network) |
| **Models** | Gemini 2.5 Pro/Flash | Any HuggingFace model | GPT-5.x |
| **Context window** | 1M tokens | Model-dependent | Model-dependent |
| **Multimodal** | Text + image | Text (typically) | Text |
| **Auth** | GCP SA key | None (cluster-internal) | OpenAI API key |
| **Data residency** | Google Cloud region | Your cluster | OpenAI servers |

## Advantages of Gemini on Vertex for OpenClaw

- **1M token context window** — Gemini 2.5 Pro supports up to 1M tokens, far more than most models. Useful for agents processing large files or long session histories.
- **Multimodal input** — can process images alongside text, enabling vision-based agent tasks.
- **No GPU provisioning** — eliminates the most expensive and complex part of the infrastructure.
- **Google Cloud SLAs** — enterprise support and availability guarantees.

## Security Considerations

- Store the SA key in an OpenShift Secret, never in ConfigMaps or images
- Use the principle of least privilege — `roles/aiplatform.user` is sufficient, avoid `roles/owner`
- Consider using [Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation) instead of SA keys for production (eliminates long-lived credentials)
- Outbound traffic goes to `*.googleapis.com` — ensure NetworkPolicy allows this

## Troubleshooting

### "Permission denied" calling Vertex AI

Check the SA has the right role:
```bash
gcloud projects get-iam-policy YOUR_PROJECT_ID \
  --filter="bindings.members:openclaw-vertex@" \
  --flatten="bindings[].members" \
  --format="table(bindings.role)"
```

### SA key file not found

Verify the secret is mounted:
```bash
oc exec deployment/openclaw -c gateway -n <namespace> -- ls -la /secrets/gcp/
```

### Wrong region

Vertex AI endpoints are region-specific. Make sure the `baseUrl` region matches where your models are available. `us-east1` and `us-central1` have the broadest model availability.

## Known Limitation: Vertex AI Auth

OpenClaw's `google-generative-ai` API adapter expects credentials in its per-agent `auth-profiles.json` store. Unlike the `anthropic-messages` adapter (which supports `apiKey: "gcp-vertex-credentials"` natively for Anthropic on Vertex AI), the Gemini adapter does not automatically resolve GCP service account credentials — even with `GOOGLE_APPLICATION_CREDENTIALS` set and the SA key mounted.

**What works:**
- `anthropic-vertex` provider with `apiKey: "gcp-vertex-credentials"` — Anthropic models (Claude) on Vertex AI
- `google` provider with `GEMINI_API_KEY` env var — Gemini models via Google AI Studio (not Vertex AI)

**What doesn't work (yet):**
- `google-vertex` provider with `api: "google-generative-ai"` and GCP SA auth — returns "No API key found for provider"

**Workaround:** Use Google AI Studio with a Gemini API key instead:
```json
{
  "agents": {
    "defaults": {
      "model": { "primary": "google/gemini-2.5-pro" }
    }
  }
}
```
Then set `GEMINI_API_KEY` in the `openclaw-secrets` Secret and run `openclaw onboard --auth-choice gemini-api-key`.

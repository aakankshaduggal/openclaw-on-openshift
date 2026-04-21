# Codex Harness on OpenShift

The [Codex Harness](https://docs.openclaw.ai/plugins/codex-harness) is a bundled OpenClaw plugin that routes agent turns through the Codex app-server instead of OpenClaw's built-in PI harness. This gives you access to Codex-native features (model discovery, thread resumption, compaction) while OpenClaw handles chat channels, session files, tools, and media delivery.

## How It Works

```
┌──────────────────────────────────────────┐
│              Pod                          │
│                                           │
│  ┌─────────────┐     ┌────────────────┐  │
│  │   gateway    │ ws  │  codex         │  │
│  │   port 18789 ├────►│  app-server    │  │
│  │              │     │  port 39175    │  │
│  └─────────────┘     └───────┬────────┘  │
│                               │           │
│                               │ HTTPS     │
│                               ▼           │
│                       ┌──────────────┐    │
│                       │ OpenAI API   │    │
│                       └──────────────┘    │
└──────────────────────────────────────────┘
```

- The **gateway** receives user messages and delegates agent turns to the Codex app-server via WebSocket (`ws://localhost:39175`)
- The **Codex app-server** runs as a sidecar in the same pod, handling the LLM interaction with OpenAI's API
- Model references like `codex/gpt-5.4` route through the harness; `openai/gpt-5.4` bypasses it entirely

## When to Use This

| Use case | Provider |
|----------|----------|
| Direct OpenAI API access | `openai/gpt-5.4` — no harness, OpenClaw PI runtime |
| Codex-managed sessions with thread resumption | `codex/gpt-5.4` — Codex harness |
| Self-hosted models via vLLM | `openai-compat/my-model` — no harness |

The Codex harness is useful when you want:
- Native Codex thread management and compaction
- Guardian-reviewed tool approvals
- Model discovery through Codex's catalog
- Mixed deployments where some agents use Codex and others use vLLM

## Important: Sidecar Image Availability

> **Status (2026-04-21):** The Codex app-server container image (`ghcr.io/openai/codex-app-server:latest`) is **not publicly accessible** — pulling it returns `403 Forbidden`. Until OpenAI publishes this image, the sidecar deployment described below will not work.
>
> **Workaround:** Use the direct OpenAI API instead of the Codex harness. Set the model to `openai/gpt-5.4` with `api: "openai-responses"` in your provider config, and provide your `OPENAI_API_KEY` via a Secret. This gives you GPT-5.x access without the sidecar, though you lose Codex-specific features (thread management, guardian approvals).

## Deploy on OpenShift

### Prerequisites

- OpenAI API key with Codex access
- Codex app-server image (`ghcr.io/openai/codex-app-server:latest`, version 0.118.0+) — see [availability note](#important-sidecar-image-availability) above

### Using the Kustomize overlay

```bash
# Copy the overlay
cp -r overlays/codex-harness overlays/my-codex

# Edit secrets with your API key
vim overlays/my-codex/codex-secret.yaml

# Edit namespace
vim overlays/my-codex/kustomization.yaml

# Deploy
oc apply -k overlays/my-codex
```

### What the overlay adds

On top of the base manifests, the Codex overlay:

1. **Adds a sidecar container** — `codex-app-server` listening on port 39175
2. **Configures WebSocket transport** — gateway connects to `ws://localhost:39175` (pod-local, no network hop)
3. **Adds secrets** — `OPENAI_API_KEY` for Codex auth, `CODEX_APP_SERVER_TOKEN` for WebSocket auth between containers
4. **Updates the ConfigMap** — enables the `codex` plugin, sets `codex/gpt-5.4` as the model, configures the embedded harness

### Resource impact

| Container | CPU request | Memory request | Memory limit |
|-----------|-------------|----------------|--------------|
| gateway | 250m | 256Mi | 1Gi |
| codex-app-server | 250m | 256Mi | 1Gi |

The Codex app-server is lightweight — it proxies requests to OpenAI's API, so compute happens remotely.

## Configuration Reference

### Model references

| Reference | What happens |
|-----------|-------------|
| `codex/gpt-5.4` | Routes through Codex app-server harness |
| `openai/gpt-5.4` | Direct OpenAI API via OpenClaw's PI runtime |
| `openai-codex/gpt-5.4` | OpenAI with OAuth, no harness |

### Plugin config fields

| Field | Default | Purpose |
|-------|---------|---------|
| `transport` | `stdio` | `stdio` spawns locally; `websocket` connects to remote |
| `url` | unset | WebSocket endpoint (e.g., `ws://localhost:39175`) |
| `authToken` | unset | Bearer token for WebSocket auth |
| `requestTimeoutMs` | `60000` | Control-plane call timeout |
| `approvalPolicy` | `never` | Tool approval policy (`never`, `on-request`) |
| `sandbox` | `workspace-write` | Sandbox mode for Codex |
| `approvalsReviewer` | `user` | Use `guardian_subagent` for automated review |

### Guardian-reviewed approvals

For production deployments where you want automated tool approval review:

```json
{
  "plugins": {
    "entries": {
      "codex": {
        "enabled": true,
        "config": {
          "appServer": {
            "transport": "websocket",
            "url": "ws://localhost:39175",
            "approvalPolicy": "on-request",
            "approvalsReviewer": "guardian_subagent",
            "sandbox": "workspace-write"
          }
        }
      }
    }
  }
}
```

## Mixed Deployments

You can run both Codex and vLLM models in the same OpenClaw instance using `runtime: "auto"`:

```json
{
  "agents": {
    "defaults": {
      "embeddedHarness": {
        "runtime": "auto"
      }
    },
    "list": [
      {
        "id": "coding-agent",
        "model": { "primary": "codex/gpt-5.4" }
      },
      {
        "id": "general-agent",
        "model": { "primary": "openai-compat/gpt-oss-20b" }
      }
    ]
  }
}
```

The `coding-agent` routes through the Codex harness; the `general-agent` uses vLLM directly.

## Troubleshooting

### Codex sidecar not starting

Check the sidecar logs:
```bash
oc logs deployment/openclaw -c codex-app-server -n <namespace>
```

Common issues:
- Missing `OPENAI_API_KEY` — check the `codex-secrets` secret
- Image pull failure — `ghcr.io/openai/codex-app-server:latest` is [not publicly available](#important-sidecar-image-availability) (returns 403). Use the direct OpenAI API fallback instead.

### Gateway can't connect to Codex

If you see WebSocket connection errors in the gateway logs:
```bash
oc logs deployment/openclaw -c gateway -n <namespace> | grep -i codex
```

- Ensure the sidecar is ready (`oc get pods` shows 2/2 or 3/3)
- Check that `url` in the config matches `ws://localhost:39175`
- Verify `CODEX_APP_SERVER_TOKEN` matches in both containers

### Protocol version mismatch

If you see "protocol rejection during handshake" errors, the Codex app-server is older than 0.118.0. Update the image tag.

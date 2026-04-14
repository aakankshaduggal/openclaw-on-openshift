# OpenClaw on OpenShift

Deploy [OpenClaw](https://github.com/openclaw/openclaw) on Red Hat OpenShift with vLLM model serving, OAuth SSO, and production-grade security — no cluster-admin required.

## Why OpenShift?

Running OpenClaw locally is fragile. On a 32GB M1 Pro MacBook:

- A 27B model consumed **41GB RAM** (262k context window KV cache) and crashed the system twice
- A 3.8B model (phi4-mini) **hallucinated fake file listings** when tool calls failed silently
- Ollama keeps idle models in memory with no automatic pressure relief

OpenShift solves all of this:

| Local Problem | OpenShift Solution |
|---------------|-------------------|
| Model exceeds laptop RAM | GPU nodes with dedicated VRAM |
| Ollama holds idle models | vLLM with autoscaling, scale-to-zero |
| Hallucinated tool results | Validated model serving with tracing |
| No auth — token-only access | OpenShift OAuth SSO |
| System crashes from resource exhaustion | Pod resource limits, workload isolation |

See [docs/local-issues.md](docs/local-issues.md) for the full analysis.

## Prerequisites

- OpenShift cluster (4.17+) with namespace-scoped access
- `oc` CLI authenticated (`oc login`)
- A model serving endpoint (vLLM, KServe, or external API)
- Block storage class (gp3-csi, managed-csi, thin-csi) — avoid NFS (SQLite requires POSIX file locking)

## Quick Start

### Option A: Using openclaw-installer (recommended)

The [openclaw-installer](https://github.com/aakankshaduggal/openclaw-installer) handles OAuth proxy, Routes, ServiceAccounts, and lifecycle management automatically.

```bash
git clone https://github.com/aakankshaduggal/openclaw-installer.git
cd openclaw-installer
npm install && npm run build && npm run dev
# Open http://localhost:3000, select OpenShift, fill in the form
```

See [docs/installer-deployment.md](docs/installer-deployment.md) for the full walkthrough.

### Option B: Kustomize

```bash
oc new-project my-openclaw

# Copy and customize the example overlay
cp -r overlays/example overlays/my-env
# Edit overlays/my-env/configmap-patch.yaml with your vLLM endpoint
# Edit overlays/my-env/kustomization.yaml with your namespace and storage class

oc apply -k overlays/my-env
```

See [overlays/example/](overlays/example/) for a complete customization example.

### Option C: Direct YAML apply

```bash
oc new-project my-openclaw

# Edit manifests/02-configmap.yaml with your model endpoint
# Edit manifests/01-secret.yaml with your gateway token

oc apply -k manifests/
```

## Architecture

```
                     +-----------------------+
                     |   OpenShift Route     |
                     |   (TLS edge)          |
                     +-----------+-----------+
                                 |
                                 v
              +------------------+------------------+
              |              Pod                     |
              |  +---------------+  +-------------+  |
              |  | oauth-proxy   |  | gateway     |  |
              |  | port 8443     +->| port 18789  |  |
              |  | (OpenShift    |  | (loopback)  |  |
              |  |  OAuth SSO)   |  |             |  |
              |  +---------------+  +------+------+  |
              |                            |          |
              |                     +------+------+   |
              |                     | PVC (5Gi)   |   |
              |                     +-------------+   |
              +--------------------------------------+
                         |
                         | OpenAI-compatible API
                         v
              +---------------------------+
              |  vLLM / KServe / API      |
              +---------------------------+
```

## Customization

The manifests use [Kustomize](https://kustomize.io/) for environment-specific configuration. The base manifests are in `manifests/`, and you create overlays to customize for your cluster.

**Common customizations:**

| What | Where |
|------|-------|
| Model endpoint URL | `overlays/<env>/configmap-patch.yaml` |
| Storage class | `overlays/<env>/kustomization.yaml` (patch) |
| Namespace | `overlays/<env>/kustomization.yaml` (`namespace:` field) |
| Gateway token | `manifests/01-secret.yaml` (or use sealed-secrets / external-secrets) |
| Resource limits | Patch `manifests/04-deployment.yaml` |
| Codex Harness | `overlays/codex-harness/` (adds Codex app-server sidecar) |

## Resources Created

| Resource | Name | Purpose |
|----------|------|---------|
| Secret | `openclaw-secrets` | Gateway token + API keys |
| ConfigMap | `openclaw-config` | Gateway configuration |
| PVC | `openclaw-state` | Persistent agent state (SQLite, memory, logs) |
| Service | `openclaw` | ClusterIP on port 18789 |
| Route | `openclaw` | TLS edge termination |
| Deployment | `openclaw` | Init container + gateway (add oauth-proxy via installer) |

> **Note:** The manual manifests provide a basic deployment. For OAuth proxy, ServiceAccount-based SSO, and lifecycle management, use the [openclaw-installer](https://github.com/aakankshaduggal/openclaw-installer).

## Security

- **restricted-v2 SCC** — non-root, random UID, no capabilities, no privilege escalation
- **OAuth proxy** (via installer) — OpenShift SSO gates all access, no raw token exposure
- **Secrets via SecretRef** — API keys never in ConfigMap or images
- **Loopback gateway** — only reachable via oauth-proxy sidecar
- **Tool deny list** — web and browser tools blocked by default
- **Recreate strategy** — avoids concurrent writer conflicts on SQLite-backed PVC

## Docs

| Document | Description |
|----------|-------------|
| [docs/local-issues.md](docs/local-issues.md) | Why local deployment fails and how OpenShift solves it |
| [docs/installer-deployment.md](docs/installer-deployment.md) | Step-by-step deployment with openclaw-installer |
| [docs/troubleshooting.md](docs/troubleshooting.md) | Common issues and fixes (route 503, model override, device pairing) |
| [docs/model-compatibility.md](docs/model-compatibility.md) | Model testing results for agentic tool-calling |
| [docs/codex-harness.md](docs/codex-harness.md) | Codex Harness plugin — sidecar deployment, mixed models, guardian approvals |

## Related Projects

- [openclaw-installer](https://github.com/aakankshaduggal/openclaw-installer) — Web-based deployment tool with OpenShift plugin
- [OpenClaw](https://github.com/openclaw/openclaw) — Upstream project
- [openclaw-operator](https://github.com/openclaw-rocks/k8s-operator) — Kubernetes operator with lifecycle management
- [openclaw-helm](https://github.com/serhanekicii/openclaw-helm) — Community Helm chart

## License

[MIT](LICENSE)

# Local Deployment Issues

> Tested: 2026-04-13 on 32GB M1 Pro MacBook

Running OpenClaw locally exposed several critical problems that reinforce the case for deploying on OpenShift/RHOAI.

## 1. Memory Exhaustion & System Instability

| Issue | Details |
|-------|---------|
| **qwen3.5:27b** (17GB on disk) | Consumed **41.9GB resident memory** due to 262k context window KV cache — more than total 32GB physical RAM |
| **Ollama default behavior** | Keeps last-used model loaded in memory for 5 min even when idle — no automatic pressure relief |
| **Impact** | System crashed to swap, fans maxed out, machine became unresponsive twice in one session |
| **Recovery** | Required Ollama restart to force-release memory; `keep_alive: 0` API call alone was insufficient |

**Lesson:** A developer laptop cannot safely run capable models (20B+) alongside normal workloads (Chrome, Slack, IDE, containers). Even "recommended" models for the hardware can silently exceed RAM once context window allocation is factored in.

## 2. Model Compatibility & Tool-Calling Failures

Not all Ollama-compatible models work with OpenClaw's agent loop:

| Model | Result |
|-------|--------|
| **qwen3.5:27b** | Tool-calling worked but destroyed system memory |
| **llama3.2:3b** | Executed tools but got confused by OpenClaw's internal system prompts (pre-compaction flush), failed to deliver clean answers |
| **phi4-mini:3.8b** | Emitted raw JSON tool-call syntax in the response body instead of using Ollama's native tool API — tools never executed |
| **phi4-mini hallucination** | When tool call failed silently, the model **fabricated an entire file listing** with realistic-looking but completely nonexistent paths — presented confidently as real data |
| **qwen2.5:7b** | Proper native tool-calling, reasonable memory footprint (~8GB) — best local option found |

**Lesson:** Model selection for agentic workloads is not just about size — it requires validated tool-calling compatibility. Small models that work fine for chat can fail dangerously for agent tasks by hallucinating tool results.

## 3. Configuration Fragility

- OpenClaw has **two layers of model config**: global (`~/.openclaw/openclaw.json`) and agent-level (`~/.openclaw/agents/main/agent/models.json`) — easy to change one and miss the other
- Gateway does **not hot-reload** config changes — requires daemon restart
- Context window sizes in config directly control memory allocation — a misconfigured `contextWindow: 262144` turned a 17GB model into a 41GB one

## Why OpenShift Solves This

| Local Problem | OpenShift Solution |
|---------------|-------------------|
| Model exceeds laptop RAM | **GPU nodes with 80GB+ VRAM** (A100/H100) — run 70B+ models without swap pressure |
| Ollama holds idle models in memory | **KServe / vLLM serving** with autoscaling — scale to zero when idle, reclaim resources automatically |
| Model compatibility is trial-and-error | **Validated model serving** via RHOAI model registry — pre-tested models with known tool-calling support |
| Hallucinated tool results go undetected | **OpenTelemetry tracing** — every tool call logged with input/output, hallucinations detectable in traces |
| Config drift between layers | **ConfigMaps + GitOps** — single source of truth, version-controlled, auditable changes |
| Developer machine becomes unusable | **Workload isolation** — agent runs in its own pod with resource limits, developer laptop stays clean |
| Security (sandbox mode off, no network policy) | **restricted-v2 SCC, NetworkPolicy, Vault secrets** — defense in depth by default |

**Bottom line:** OpenClaw is a compelling agent framework, but running it locally is fragile, resource-dangerous, and unobservable. OpenShift + RHOAI provides the resource headroom, isolation, and operational guardrails that make it production-viable.

# Troubleshooting

## Route returns "Application is not available" (503)

**Cause:** The oauth-proxy uses `--http-address` (plain HTTP) by default. If the Route has `tls.termination: reencrypt`, the router expects TLS on the backend and the connection fails silently.

**Fix:** Recreate the Route with `edge` termination:

```bash
oc delete route openclaw -n <namespace>

cat <<'EOF' | oc apply -f -
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: openclaw
  namespace: <namespace>
  labels:
    app: openclaw
  annotations:
    haproxy.router.openshift.io/timeout: 30m
spec:
  to:
    kind: Service
    name: openclaw
    weight: 100
  port:
    targetPort: oauth-ui
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
EOF
```

**Verification:** `curl -s -o /dev/null -w "%{http_code}" -k https://<route-url>` should return `403` (OAuth redirect), not `503`.

## Gateway uses wrong model / "No API key" errors

**Cause:** OpenClaw auto-generates its config on first start. If the gateway detects a provider (e.g., Anthropic) it will override the ConfigMap settings with its own defaults. This means the ConfigMap says vLLM but the gateway is actually trying to use Anthropic.

**Symptoms in logs:**
```
No API key found for provider "anthropic"
model fallback decision: decision=candidate_failed requested=anthropic/claude-sonnet-4-6
```

**Fix:** Patch the ConfigMap with the correct model provider config, then restart:

```bash
# Export current config
oc get configmap openclaw-config -n <namespace> \
  -o jsonpath='{.data.openclaw\.json}' > /tmp/openclaw-config.json

# Edit /tmp/openclaw-config.json:
# - Set agents.defaults.model.primary to "openai-compat/gpt-oss-20b"
# - Add models.providers.openai-compat with your vLLM endpoint

# Apply and restart
oc create configmap openclaw-config \
  --from-file=openclaw.json=/tmp/openclaw-config.json \
  -n <namespace> --dry-run=client -o yaml | oc apply -f -
oc rollout restart deployment/openclaw -n <namespace>
```

**Verification:**
```bash
oc logs deployment/openclaw -c gateway -n <namespace> | grep "agent model"
# Should show: agent model: openai-compat/gpt-oss-20b
```

## Device pairing required after SSO login

**Cause:** The Control UI browser session needs a one-time device approval, even after OpenShift OAuth login.

**Fix:**
```bash
# Get the request ID from the UI prompt, then:
oc exec deployment/openclaw -n <namespace> -c gateway -- \
  openclaw devices approve <request-id>
```

Alternatively, use the **Open** action from the installer's **Instances** tab — it opens with the token pre-filled and may auto-pair.

## Pod stuck in CrashLoopBackOff

**Cause:** OpenClaw auto-generates config at startup. If the config on the PVC conflicts with the ConfigMap, the gateway detects a change, overwrites the file, and triggers a process restart that kills PID 1.

**Fix:** Delete the PVC data and redeploy:
```bash
oc scale deployment/openclaw --replicas=0 -n <namespace>
oc delete pvc openclaw-home-pvc -n <namespace>
# Redeploy via installer or re-apply manifests
```

## Useful diagnostic commands

```bash
# Pod status
oc get pods -n <namespace>

# Gateway logs
oc logs deployment/openclaw -c gateway -n <namespace>

# OAuth proxy logs
oc logs deployment/openclaw -c oauth-proxy -n <namespace>

# Check route config
oc get route openclaw -n <namespace> -o yaml

# Check what model is configured
oc get configmap openclaw-config -n <namespace> \
  -o jsonpath='{.data.openclaw\.json}' | python3 -m json.tool

# Exec into the gateway
oc exec -it deployment/openclaw -c gateway -n <namespace> -- sh
```

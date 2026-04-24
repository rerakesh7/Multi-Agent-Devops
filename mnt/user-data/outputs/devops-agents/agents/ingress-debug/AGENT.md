# 🔥 Agent-002: High-Traffic Ingress 502 Debugging Agent
# Clouds: GCP (GKE + GLB) · AWS (EKS + ALB/NLB) · Azure (AKS + App Gateway)
# HITL: Required for production config changes

```
╔══════════════════════════════════════════════════════════════════╗
║           INGRESS 502 DEBUGGER — REAL-TIME PLAYBOOK              ║
╚══════════════════════════════════════════════════════════════════╝

502 BAD GATEWAY FLOW:

  Client                                                      Pod
    │                                                          │
    │──── HTTPS ──▶[Cloud LB]──▶[Ingress Controller]──▶[Service]──▶│
    │                  │               │                   │        │
    │◀── 502 ──────────┤               │                   │        │
    │                  │               │                   │        │
    │            WHERE IS THE 502?     │                   │        │
    │                  │               │                   │        │
    │         ┌────────┴─────┐   ┌─────┴──────┐  ┌────────┴──┐    │
    │         │  Upstream    │   │  NGINX/    │  │  Pod Not  │    │
    │         │  Timeout     │   │  Envoy Err │  │  Ready    │    │
    │         │  (504→502)   │   │            │  │           │    │
    │         └──────────────┘   └────────────┘  └───────────┘    │
    │                                                               │
    │  ROOT CAUSES:                                                 │
    │  1. Pod crash/OOMKill → endpoints removed                    │
    │  2. Readiness probe failing → pod removed from LB            │
    │  3. Upstream timeout too short vs actual response time       │
    │  4. Keep-alive mismatch (LB closes before nginx)             │
    │  5. SSL/TLS termination misconfiguration                     │
    │  6. Max connections exceeded (worker_connections)            │
```

---

## Skill: 502 Rapid Triage Playbook

### Step 1: Determine 502 Origin (< 2 min)

```bash
#!/bin/bash
# 502-triage.sh — Run this FIRST during a 502 incident

NS="${1:-default}"
SVC="${2:-my-service}"

echo "══════════════════════════════════════"
echo " 502 TRIAGE — $(date -u)"
echo " Namespace: $NS | Service: $SVC"
echo "══════════════════════════════════════"

# 1. Check pod health immediately
echo ""
echo "▶ [1/6] Pod Status"
kubectl get pods -n "$NS" -l "app=$SVC" \
  --sort-by='.status.conditions[0].lastTransitionTime' \
  -o wide

# 2. Recent events (last 10 min)
echo ""
echo "▶ [2/6] Recent Events (Warning only)"
kubectl get events -n "$NS" \
  --field-selector reason=BackOff,reason=OOMKilling,reason=Failed,reason=Unhealthy \
  --sort-by='.lastTimestamp' | tail -20

# 3. Endpoint health
echo ""
echo "▶ [3/6] Endpoint Readiness"
kubectl get endpoints -n "$NS" "$SVC" -o json | \
  jq '{
    ready: .subsets[].addresses | length,
    not_ready: (.subsets[].notReadyAddresses // []) | length
  }'

# 4. Ingress config
echo ""
echo "▶ [4/6] Ingress Config"
kubectl get ingress -n "$NS" -o yaml | \
  grep -E "host|path|backend|timeout|proxy"

# 5. Recent restarts
echo ""
echo "▶ [5/6] Container Restart Counts"
kubectl get pods -n "$NS" -l "app=$SVC" \
  -o jsonpath='{range .items[*]}{.metadata.name}: restarts={.status.containerStatuses[0].restartCount}, state={.status.containerStatuses[0].state}{"\n"}{end}'

# 6. Resource pressure
echo ""
echo "▶ [6/6] Resource Usage (top)"
kubectl top pods -n "$NS" -l "app=$SVC" 2>/dev/null || echo "  metrics-server not available"

echo ""
echo "══════════ TRIAGE COMPLETE ══════════"
echo "Review above before making any changes"
echo "[HUMAN REVIEW] Required for prod fixes"
```

---

### Step 2: Root Cause Playbooks

#### RCA-1: Pod Not Ready (Most Common)
```bash
# Diagnosis
POD=$(kubectl get pod -n $NS -l app=$SVC -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $POD -n $NS | grep -A 10 "Readiness\|Liveness\|Events"
kubectl logs $POD -n $NS --previous --tail=50   # Check crash logs

# HITL Gate ─────────────────────────────────────────────────
# ⚠️ BEFORE applying fix, present to human:
# "Readiness probe is failing with HTTP 500 on /health"
# "Fix: Update readiness probe initial delay from 5s to 30s"
# "Impact: 0 downtime — pods will not receive traffic during startup"
# [APPROVE] [REJECT]
# ────────────────────────────────────────────────────────────

# Fix: Patch readiness probe (after approval)
kubectl patch deployment $SVC -n $NS --type='json' -p='[
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/0/readinessProbe/initialDelaySeconds",
    "value": 30
  }
]'
```

#### RCA-2: Upstream Timeout (GKE/NGINX)
```bash
# Diagnosis: Check NGINX upstream timeout settings
kubectl exec -n ingress-nginx $(kubectl get pod -n ingress-nginx -o name | head -1) \
  -- nginx -T 2>/dev/null | grep -E "proxy_read_timeout|proxy_connect_timeout|upstream"

# NGINX Ingress annotation fix (after HITL approval)
# For ingress object:
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: $SVC
  namespace: $NS
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
    nginx.ingress.kubernetes.io/upstream-keepalive-timeout: "75"
    nginx.ingress.kubernetes.io/upstream-keepalive-connections: "32"
EOF
```

#### RCA-3: AWS ALB 502 Specific
```bash
# ALB 502s are usually target health failures
# Check target group health
aws elbv2 describe-target-health \
  --target-group-arn $TG_ARN \
  --query 'TargetHealthDescriptions[?TargetHealth.State!=`healthy`]'

# Common ALB 502 cause: deregistration_delay too short
# HITL gate before change:
aws elbv2 modify-target-group-attributes \
  --target-group-arn $TG_ARN \
  --attributes Key=deregistration_delay.timeout_seconds,Value=60

# ALB access logs analysis
aws s3 cp s3://$ALB_LOG_BUCKET/$(date +%Y/%m/%d)/ /tmp/alb-logs/ --recursive
zcat /tmp/alb-logs/*.log.gz | awk '{print $9}' | sort | uniq -c | sort -rn | head -20
# Col 9 = target_status_code — look for 5xx spikes
```

#### RCA-4: Azure App Gateway 502
```bash
# Azure: Check backend health
az network application-gateway show-backend-health \
  --resource-group $RG \
  --name $AGW_NAME \
  --query 'backendAddressPools[].backendHttpSettingsCollection[].servers[?health!=`Healthy`]'

# Check App Gateway logs
az monitor activity-log list \
  --resource-group $RG \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --query '[?contains(operationName.value, `applicationGateways`)]'
```

---

### Step 3: High-Traffic 502 Prevention

```yaml
# GKE/EKS/AKS — Production ingress with 502-prevention annotations
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production-ingress
  annotations:
    # Prevent 502s during pod restarts
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    
    # Keep-alive (CRITICAL: must be > upstream keep-alive timeout)
    nginx.ingress.kubernetes.io/upstream-keepalive-timeout: "75"
    nginx.ingress.kubernetes.io/upstream-keepalive-connections: "100"
    
    # Circuit breaker
    nginx.ingress.kubernetes.io/server-snippet: |
      location / {
        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 3;
        proxy_next_upstream_timeout 10s;
      }
    
    # Rate limiting to protect upstream
    nginx.ingress.kubernetes.io/rate-limit: "1000"
    nginx.ingress.kubernetes.io/rate-limit-burst-multiplier: "5"
    
    # Health check path
    nginx.ingress.kubernetes.io/health-check-path: "/health"
    nginx.ingress.kubernetes.io/health-check-interval-seconds: "5"
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

---

### 502 Dashboard Queries (Prometheus/Grafana)

```promql
# 502 rate by service (last 5 min)
rate(nginx_ingress_controller_requests{status="502"}[5m])

# 502 percentage of total traffic
sum(rate(nginx_ingress_controller_requests{status="502"}[5m])) 
  / sum(rate(nginx_ingress_controller_requests[5m])) * 100

# Upstream latency P99 (find slow upstreams causing timeouts)
histogram_quantile(0.99, 
  rate(nginx_ingress_controller_response_duration_seconds_bucket[5m])
)

# Alert: 502 rate > 1%
- alert: HighIngressErrorRate
  expr: |
    sum(rate(nginx_ingress_controller_requests{status=~"5.."}[5m])) 
    / sum(rate(nginx_ingress_controller_requests[5m])) > 0.01
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "High 5xx rate on ingress"
    runbook: "https://wiki.internal/runbooks/502-debug"
```

---

## HITL Approval Template for 502 Fixes

```
┌────────────────────────────────────────────────────────┐
│ ⚠️  PRODUCTION INGRESS CHANGE — APPROVAL REQUIRED        │
├────────────────────────────────────────────────────────┤
│ Incident ID  : INC-{ID}                               │
│ 502 Rate     : {RATE}% of traffic                     │
│ Duration     : {DURATION} minutes                      │
│ Affected SVC : {SERVICE} in {NAMESPACE}               │
├────────────────────────────────────────────────────────┤
│ PROPOSED FIX :                                        │
│   Change proxy-read-timeout: 60s → 300s              │
│                                                        │
│ RISK         : LOW — annotation change only           │
│ ROLLBACK     : Revert annotation (30s)                │
│ EXPECTED RES : 502 rate drops to 0% within 60s        │
├────────────────────────────────────────────────────────┤
│ [✅ APPROVE]  [❌ REJECT]  [🔄 MODIFY FIRST]           │
└────────────────────────────────────────────────────────┘
```

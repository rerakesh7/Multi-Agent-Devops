# 🚀 Agent-003: Zero-Downtime Deployments Agent
# Targets: GKE · EKS · AKS
# HITL: ALWAYS required for production deployments

```
╔══════════════════════════════════════════════════════════════════════╗
║          ZERO-DOWNTIME DEPLOYMENT AGENT v2.0                         ║
║  Strategy: Blue/Green · Canary · Rolling · Feature Flags             ║
║  ⚠️  ALL PRODUCTION DEPLOYS REQUIRE HUMAN SIGN-OFF                    ║
╚══════════════════════════════════════════════════════════════════════╝

DEPLOYMENT DECISION TREE:

  Is this a breaking DB schema change?
       │
       ├── YES ──▶ Blue/Green Deploy (mandatory)
       │           + Expand-Contract migration pattern
       │
       └── NO ──▶ Is risk HIGH or traffic > 10k rpm?
                  │
                  ├── YES ──▶ Canary Deploy (1% → 10% → 50% → 100%)
                  │           with automated rollback triggers
                  │
                  └── NO ──▶ Rolling Deploy
                              with PodDisruptionBudget
```

---

## Skill: Blue/Green Deployment (All 3 Clouds)

```yaml
# blue-green-deploy.yaml
# Step 1: Deploy GREEN (new version) alongside BLUE (current)
# Step 2: Smoke test GREEN
# Step 3: [HUMAN APPROVAL] switch traffic
# Step 4: Keep BLUE for 30m, then teardown

---
# GREEN deployment (new version — receives 0% traffic initially)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP}-green
  namespace: ${NS}
  labels:
    app: ${APP}
    slot: green
    version: "${NEW_VERSION}"
spec:
  replicas: ${REPLICAS}
  selector:
    matchLabels:
      app: ${APP}
      slot: green
  template:
    metadata:
      labels:
        app: ${APP}
        slot: green
        version: "${NEW_VERSION}"
    spec:
      # Anti-affinity: spread across zones
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: ${APP}
            slot: green
      containers:
      - name: ${APP}
        image: "${IMAGE}:${NEW_VERSION}"
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]   # Drain connections
---
# Service initially pointing to BLUE
apiVersion: v1
kind: Service
metadata:
  name: ${APP}
  namespace: ${NS}
spec:
  selector:
    app: ${APP}
    slot: blue              # ← Switch to 'green' after approval
  ports:
  - port: 80
    targetPort: 8080
```

### Blue/Green Switch Script
```bash
#!/bin/bash
# bg-switch.sh — Execute ONLY after HITL approval

APP=$1
NS=$2
TARGET_SLOT=$3   # blue or green

echo "══════════════════════════════════"
echo " Blue/Green Traffic Switch"
echo " App: $APP | Namespace: $NS"
echo " Switching traffic to: $TARGET_SLOT"
echo "══════════════════════════════════"

# Pre-switch health check
GREEN_READY=$(kubectl get pods -n $NS -l "app=$APP,slot=$TARGET_SLOT" \
  -o jsonpath='{.items[*].status.conditions[?(@.type=="Ready")].status}' | tr ' ' '\n' | grep -c True)
GREEN_TOTAL=$(kubectl get pods -n $NS -l "app=$APP,slot=$TARGET_SLOT" --no-headers | wc -l)

echo "Ready: $GREEN_READY/$GREEN_TOTAL pods"
if [ "$GREEN_READY" -ne "$GREEN_TOTAL" ]; then
  echo "❌ ABORT: Not all pods ready. Cannot switch traffic."
  exit 1
fi

# Run smoke tests against green before switching
echo "Running smoke tests..."
GREEN_SVC_IP=$(kubectl get svc -n $NS ${APP}-${TARGET_SLOT}-internal -o jsonpath='{.spec.clusterIP}')
HTTP_STATUS=$(kubectl run -it --rm smoke-test --image=curlimages/curl --restart=Never -- \
  curl -o /dev/null -s -w "%{http_code}" http://${GREEN_SVC_IP}/health 2>/dev/null)

if [ "$HTTP_STATUS" != "200" ]; then
  echo "❌ ABORT: Smoke test failed (HTTP $HTTP_STATUS)"
  exit 1
fi
echo "✅ Smoke tests passed"

# ─── HUMAN APPROVAL GATE ────────────────────────────────────
echo ""
echo "╔══════════════════════════════════════════════════════╗"
echo "║  ⚠️  APPROVAL REQUIRED TO SWITCH PRODUCTION TRAFFIC   ║"
echo "║                                                        ║"
echo "║  From: $(kubectl get svc $APP -n $NS -o jsonpath='{.spec.selector.slot}')  →  To: $TARGET_SLOT                       ║"
echo "║  Impact: All production traffic affected              ║"
echo "║  Rollback: Re-run with original slot (< 30 seconds)  ║"
echo "╚══════════════════════════════════════════════════════╝"
read -p "Type 'APPROVE' to proceed: " CONFIRM
if [ "$CONFIRM" != "APPROVE" ]; then
  echo "❌ Cancelled by operator"
  exit 0
fi
# ────────────────────────────────────────────────────────────

# Execute switch
kubectl patch service $APP -n $NS \
  -p "{\"spec\":{\"selector\":{\"app\":\"$APP\",\"slot\":\"$TARGET_SLOT\"}}}"

echo "✅ Traffic switched to $TARGET_SLOT"
echo "Monitoring for 60 seconds..."

# Post-switch validation (60s window)
for i in $(seq 1 12); do
  sleep 5
  ERROR_RATE=$(kubectl exec -n monitoring prometheus-0 -- \
    wget -qO- "http://localhost:9090/api/v1/query?query=rate(http_requests_total{status=~'5..',service='$APP'}[1m])" 2>/dev/null | \
    jq '.data.result[0].value[1] // "0"')
  echo "  [${i}0s] 5xx rate: $ERROR_RATE rps"
  
  if (( $(echo "$ERROR_RATE > 0.1" | bc -l) )); then
    echo "❌ High error rate detected! Auto-rolling back..."
    kubectl patch service $APP -n $NS \
      -p "{\"spec\":{\"selector\":{\"app\":\"$APP\",\"slot\":\"blue\"}}}"
    echo "✅ Rolled back to blue"
    exit 1
  fi
done

echo "✅ Deployment stable. BLUE slot will be kept for 30m then removed."
```

---

## Skill: Canary Deployment (Argo Rollouts / Native)

```yaml
# canary-rollout.yaml — Argo Rollouts CRD
# Requires: kubectl argo rollouts plugin
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: ${APP}
  namespace: ${NS}
spec:
  replicas: ${REPLICAS:-10}
  selector:
    matchLabels:
      app: ${APP}
  template:
    metadata:
      labels:
        app: ${APP}
    spec:
      containers:
      - name: ${APP}
        image: "${IMAGE}:${VERSION}"
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
  strategy:
    canary:
      # Automated analysis — will pause at each step for HITL
      analysis:
        templates:
        - templateName: success-rate
        startingStep: 2    # Start analysis at step 2 (10%)
        args:
        - name: service-name
          value: ${APP}-canary
      steps:
      # ── Step 1: 5% canary ──────────────────────────────────
      - setWeight: 5
      - pause:
          duration: 60s       # Wait 60s then continue auto
      # ── Step 2: 20% + analysis starts ─────────────────────
      - setWeight: 20
      - pause: {}             # {} = WAIT FOR HUMAN APPROVAL
      # ── Step 3: 50% ────────────────────────────────────────
      - setWeight: 50
      - pause:
          duration: 300s      # 5 min auto-pause
      # ── Step 4: 100% ───────────────────────────────────────
      - setWeight: 100
      
      # Automatic rollback triggers
      canaryService: ${APP}-canary
      stableService: ${APP}-stable
      trafficRouting:
        nginx:                # or: alb | istio | smi
          stableIngress: ${APP}-stable
          annotationPrefix: nginx.ingress.kubernetes.io
          additionalIngressAnnotations:
            canary-by-header: X-Canary
            canary-by-header-value: "true"
---
# Analysis Template — defines rollback criteria
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: ${NS}
spec:
  args:
  - name: service-name
  metrics:
  - name: success-rate
    interval: 30s
    successCondition: result[0] >= 0.99     # 99% success rate
    failureLimit: 3                          # 3 failures = auto-rollback
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{service="{{args.service-name}}",status!~"5.."}[5m]))
          /
          sum(rate(http_requests_total{service="{{args.service-name}}"}[5m]))
```

---

## Skill: PodDisruptionBudget (Rolling Deploy Safety)

```yaml
# pdb.yaml — Ensure minimum availability during rolling updates
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: ${APP}-pdb
  namespace: ${NS}
spec:
  # Guarantee at least 80% pods always available
  minAvailable: "80%"
  selector:
    matchLabels:
      app: ${APP}
---
# Deployment rolling update config
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP}
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0       # Never take pods offline first
      maxSurge: 25%           # Bring new pods up before removing old
  minReadySeconds: 15         # Wait 15s after ready before continuing
  progressDeadlineSeconds: 600
```

---

## Cloud-Specific Deploy Patterns

### GKE: Binary Authorization
```bash
# GKE enforces image signing before deploy
gcloud container binauthz policy import policy.yaml --project=$GCP_PROJECT

# policy.yaml — only allow signed images
cat > policy.yaml <<EOF
globalPolicyEvaluationMode: ENABLE
defaultAdmissionRule:
  evaluationMode: REQUIRE_ATTESTATION
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
  requireAttestationsBy:
  - projects/${GCP_PROJECT}/attestors/production-attestor
EOF
```

### EKS: Deployment with OPA Gatekeeper
```bash
# Pre-deploy validation (runs before any EKS deployment)
kubectl apply --dry-run=server -f deployment.yaml
# OPA Gatekeeper will validate against ConstraintTemplates
```

### AKS: Deployment with Azure Policy
```bash
# AKS enforces Azure Policy at admission
az aks enable-addons \
  --resource-group $RG \
  --name $AKS_CLUSTER \
  --addons azure-policy
```

---

## HITL Production Deploy Checklist

```
┌─────────────────────────────────────────────────────────────┐
│  PRE-DEPLOYMENT CHECKLIST — Human Must Verify All Items      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Application                                                 │
│  □ All tests passing in CI (link to run)                    │
│  □ Image scanned — no CRITICAL CVEs                         │
│  □ Image signed (Binary Auth / Notary)                      │
│  □ Config diff reviewed (no secrets in env vars)            │
│                                                              │
│  Infrastructure                                              │
│  □ PDB in place (minAvailable confirmed)                    │
│  □ HPA configured (min/max replicas set)                    │
│  □ Resources limits set on all containers                   │
│  □ Readiness/liveness probes configured                     │
│                                                              │
│  Database (if schema change)                                 │
│  □ Migration is backward-compatible                         │
│  □ Rollback migration script tested                         │
│  □ DB backup taken (timestamp: _______)                     │
│                                                              │
│  Monitoring                                                  │
│  □ Dashboards open during deploy                            │
│  □ Alerts muted for 15 min (not longer)                     │
│  □ On-call engineer aware                                   │
│                                                              │
│  Approved by: ____________  Time: __________                │
│  [START DEPLOY]  [ABORT]                                    │
└─────────────────────────────────────────────────────────────┘
```

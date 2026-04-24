# 📚 Shared Skills Library
# Reusable across all agents

---

## skills/hitl-gate.sh — Universal Human-in-the-Loop Gate
```bash
#!/bin/bash
# hitl-gate.sh — Call this before ANY production action
# Usage: source hitl-gate.sh && require_approval "action description" "env" "risk"

require_approval() {
  local ACTION="$1"
  local ENV="${2:-unknown}"
  local RISK="${3:-MEDIUM}"
  local CORRELATION_ID=$(cat /proc/sys/kernel/random/uuid 2>/dev/null || uuidgen)
  
  # Auto-approve for dev/non-prod
  if [[ "$ENV" == "dev" || "$ENV" == "development" || "$ENV" == "local" ]]; then
    echo "✅ Auto-approved (dev environment) | CID: $CORRELATION_ID"
    export HITL_APPROVED=true
    export HITL_CORRELATION_ID=$CORRELATION_ID
    return 0
  fi
  
  # Slack notification for staging
  if [[ "$ENV" == "staging" ]]; then
    echo "📣 Notifying Slack (staging deploy)..."
    notify_slack "$ACTION" "$ENV" "$CORRELATION_ID"
    export HITL_APPROVED=true
    return 0
  fi
  
  # Manual approval for prod
  echo ""
  echo "┌──────────────────────────────────────────────────────────┐"
  echo "│  ⚠️  PRODUCTION ACTION — MANUAL APPROVAL REQUIRED          │"
  echo "├──────────────────────────────────────────────────────────┤"
  printf "│  Correlation ID : %-40s │\n" "$CORRELATION_ID"
  printf "│  Environment    : %-40s │\n" "$ENV"
  printf "│  Risk Level     : %-40s │\n" "$RISK"
  printf "│  Action         : %-40s │\n" "${ACTION:0:40}"
  printf "│  Requested by   : %-40s │\n" "$(whoami)@$(hostname)"
  printf "│  Timestamp      : %-40s │\n" "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
  echo "├──────────────────────────────────────────────────────────┤"
  echo "│                                                            │"
  echo "│  Log this approval in: #platform-approvals (Slack)       │"
  echo "│                                                            │"
  echo "└──────────────────────────────────────────────────────────┘"
  
  # Two-person rule for CRITICAL risk
  if [[ "$RISK" == "CRITICAL" ]]; then
    read -p "Approver 1 — Type 'APPROVE' to continue: " APPROVE1
    read -p "Approver 2 — Type 'APPROVE' to confirm: " APPROVE2
    if [[ "$APPROVE1" != "APPROVE" || "$APPROVE2" != "APPROVE" ]]; then
      echo "❌ Two-person approval not satisfied. Aborted."
      export HITL_APPROVED=false
      return 1
    fi
  else
    read -p "Type 'APPROVE' to proceed or anything else to abort: " CONFIRM
    if [[ "$CONFIRM" != "APPROVE" ]]; then
      echo "❌ Cancelled by operator"
      export HITL_APPROVED=false
      return 1
    fi
  fi
  
  export HITL_APPROVED=true
  export HITL_CORRELATION_ID=$CORRELATION_ID
  echo "✅ Approved | CID: $CORRELATION_ID | Approver: $(whoami)"
  
  # Audit log
  echo "$(date -u) | CID=$CORRELATION_ID | ENV=$ENV | RISK=$RISK | ACTION=$ACTION | APPROVER=$(whoami)" \
    >> /var/log/devops-agent-approvals.log 2>/dev/null || true
}

notify_slack() {
  local MSG="$1"
  local ENV="$2"
  local CID="$3"
  
  [[ -z "$SLACK_WEBHOOK_URL" ]] && return 0
  
  curl -s -X POST "$SLACK_WEBHOOK_URL" \
    -H 'Content-type: application/json' \
    -d "{
      \"text\": \"🚀 Deploy to *$ENV*: $MSG\",
      \"attachments\": [{
        \"color\": \"warning\",
        \"fields\": [
          {\"title\": \"Correlation ID\", \"value\": \"$CID\", \"short\": true},
          {\"title\": \"Operator\", \"value\": \"$(whoami)\", \"short\": true}
        ]
      }]
    }" || true
}
```

---

## skills/health-check.sh — Post-Action Health Verification
```bash
#!/bin/bash
# health-check.sh — Run after any deployment/change

health_check() {
  local NS=$1
  local SVC=$2
  local TIMEOUT=${3:-120}
  local ENDPOINT=${4:-"/health"}
  
  echo "══ Health Check: $SVC ($NS) ══"
  
  local START=$(date +%s)
  local PASSED=0
  
  while [ $(( $(date +%s) - START )) -lt $TIMEOUT ]; do
    # Pod readiness
    READY=$(kubectl get deployment "$SVC" -n "$NS" \
      -o jsonpath='{.status.readyReplicas}' 2>/dev/null || echo "0")
    DESIRED=$(kubectl get deployment "$SVC" -n "$NS" \
      -o jsonpath='{.spec.replicas}' 2>/dev/null || echo "1")
    
    if [ "$READY" = "$DESIRED" ] && [ "$READY" != "0" ]; then
      echo "✅ Pods: $READY/$DESIRED ready"
      PASSED=1
      break
    fi
    
    echo "⏳ Waiting... ($READY/$DESIRED pods ready)"
    sleep 10
  done
  
  if [ "$PASSED" = "0" ]; then
    echo "❌ Health check FAILED after ${TIMEOUT}s"
    return 1
  fi
  
  # HTTP health check
  SVC_IP=$(kubectl get svc "$SVC" -n "$NS" -o jsonpath='{.spec.clusterIP}')
  HTTP_STATUS=$(kubectl run -it --rm health-probe-$(date +%s) \
    --image=curlimages/curl:8.10.1 \
    --restart=Never -- \
    curl -s -o /dev/null -w "%{http_code}" \
    "http://${SVC_IP}${ENDPOINT}" 2>/dev/null || echo "000")
  
  if [ "$HTTP_STATUS" = "200" ]; then
    echo "✅ HTTP health: $HTTP_STATUS"
    return 0
  else
    echo "❌ HTTP health: $HTTP_STATUS (expected 200)"
    return 1
  fi
}
```

---

## skills/rollback.sh — Universal Rollback Handler
```bash
#!/bin/bash
# rollback.sh — Immediate rollback for any deployment type

rollback() {
  local TYPE=$1    # helm | kubectl | terraform
  local NAME=$2
  local NS=${3:-default}
  
  source hitl-gate.sh
  require_approval "ROLLBACK: $TYPE/$NAME in $NS" "$NS" "HIGH"
  
  [[ "$HITL_APPROVED" != "true" ]] && return 1
  
  case "$TYPE" in
    helm)
      echo "Rolling back Helm release: $NAME"
      helm rollback "$NAME" -n "$NS" --wait --timeout 5m
      ;;
    kubectl)
      echo "Rolling back Kubernetes deployment: $NAME"
      kubectl rollout undo deployment/"$NAME" -n "$NS"
      kubectl rollout status deployment/"$NAME" -n "$NS" --timeout=5m
      ;;
    terraform)
      echo "⚠️  Terraform rollback requires re-running previous plan"
      echo "Check git history for last known-good state"
      echo "Run: git log terraform/ | head -10"
      ;;
  esac
  
  health_check "$NS" "$NAME"
}
```

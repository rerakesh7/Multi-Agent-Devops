# 🔴 Agent-008: Production-Grade Agent
# Real Debugging Playbooks · ASCII Architecture · Incident Response
# HITL: ALWAYS for production actions | 2-person rule for P0

```
╔══════════════════════════════════════════════════════════════════╗
║           PRODUCTION-GRADE AGENT v2.0                            ║
║   Incident Response · Real Playbooks · Live Diagnostics          ║
║   ⚠️  P0 INCIDENTS: 2-PERSON APPROVAL RULE                        ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🏛️ Full Production Architecture (ASCII)

```
╔═══════════════════════════════════════════════════════════════════════════════════╗
║                     PRODUCTION MULTI-CLOUD ARCHITECTURE                           ║
╚═══════════════════════════════════════════════════════════════════════════════════╝

                              GLOBAL TRAFFIC LAYER
    ┌──────────────────────────────────────────────────────────────────────────┐
    │  Cloudflare (DDoS, WAF, TLS termination)                                 │
    │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                     │
    │  │ CF Workers  │  │  Rate Limit │  │  Geo Routing│                     │
    │  └──────┬──────┘  └─────────────┘  └──────┬──────┘                     │
    └─────────┼────────────────────────────────────┼──────────────────────────┘
              │                                    │
    ┌─────────▼────────────────────────────────────▼──────────────────────────┐
    │                       LOAD BALANCER TIER                                 │
    │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────────┐  │
    │  │  GCP GLB     │  │  AWS ALB     │  │     Azure App Gateway v2     │  │
    │  │  (US+EU)     │  │  (US-EAST)   │  │         (EUROPE)             │  │
    │  └──────┬───────┘  └──────┬───────┘  └──────────────┬───────────────┘  │
    └─────────┼─────────────────┼──────────────────────────┼──────────────────┘
              │                 │                          │
    ┌─────────▼───────┐  ┌──────▼───────┐  ┌─────────────▼─────────────────┐
    │  GKE CLUSTER    │  │  EKS CLUSTER │  │        AKS CLUSTER             │
    │  us-central1    │  │  us-east-1   │  │           eastus               │
    │                 │  │              │  │                                 │
    │  ┌───────────┐  │  │ ┌──────────┐ │  │  ┌───────────────────────────┐ │
    │  │NGINX Ingr │  │  │ │ AWS LBC  │ │  │  │   AGIC (Ingress Ctrl)     │ │
    │  └─────┬─────┘  │  │ └────┬─────┘ │  │  └─────────────┬─────────────┘ │
    │        │        │  │      │        │  │                │               │
    │  ┌─────▼─────┐  │  │ ┌────▼─────┐ │  │  ┌─────────────▼────────────┐  │
    │  │ api-svc   │  │  │ │ api-svc  │ │  │  │         api-svc           │  │
    │  │ web-svc   │  │  │ │ web-svc  │ │  │  │         web-svc           │  │
    │  │ worker    │  │  │ │ worker   │ │  │  │         worker            │  │
    │  └─────┬─────┘  │  │ └────┬─────┘ │  │  └─────────────┬────────────┘  │
    └────────┼────────┘  └──────┼────────┘  └────────────────┼───────────────┘
             │                  │                             │
    ┌────────▼──────────────────▼─────────────────────────────▼──────────────┐
    │                       DATA TIER (Multi-Region)                          │
    │                                                                          │
    │  ┌─────────────────────────────────────────────────────────────────┐   │
    │  │                    PRIMARY DATABASE                               │   │
    │  │  GCP CloudSQL (us-central1) ──────── REPLICA ─────▶ us-east1   │   │
    │  │  AWS RDS Multi-AZ (us-east-1)                                   │   │
    │  │  Azure SQL Geo-Redundant (eastus ──▶ westeurope)                │   │
    │  └─────────────────────────────────────────────────────────────────┘   │
    │                                                                          │
    │  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────────────┐  │
    │  │ Redis (Memstore) │  │  Pub/Sub / SQS   │  │  Object Storage        │  │
    │  │ Elasticache      │  │  Service Bus     │  │  GCS/S3/Azure Blob     │  │
    │  │ Azure Cache      │  │  (async jobs)    │  │  (static assets/data)  │  │
    │  └─────────────────┘  └─────────────────┘  └────────────────────────┘  │
    └──────────────────────────────────────────────────────────────────────────┘

    ┌──────────────────────────────────────────────────────────────────────────┐
    │                    OBSERVABILITY STACK                                    │
    │                                                                           │
    │  Metrics    : Prometheus + Grafana Cloud                                  │
    │  Logging    : Fluentd → GCS/S3 → BigQuery/Athena + Kibana                │
    │  Tracing    : OpenTelemetry → Jaeger/Google Cloud Trace                   │
    │  Alerts     : Alertmanager → PagerDuty → Slack                           │
    │  Uptime     : Synthetic monitoring (every 30s from 5 regions)             │
    └──────────────────────────────────────────────────────────────────────────┘

    ┌──────────────────────────────────────────────────────────────────────────┐
    │                    PLATFORM CONTROL PLANE                                 │
    │                                                                           │
    │  GitOps     : ArgoCD (per cluster) + ApplicationSets                      │
    │  Secrets    : HashiCorp Vault (with k8s auth backend)                     │
    │  Policy     : OPA Gatekeeper (admission webhooks)                         │
    │  Security   : Falco (runtime threat detection)                            │
    │  Cost       : Kubecost (per-namespace breakdown)                          │
    └──────────────────────────────────────────────────────────────────────────┘
```

---

## 🚨 Real Debugging Playbooks

### Playbook-001: Production Service Down (P0)

```bash
#!/bin/bash
# p0-response.sh — Immediate P0 response (first 5 minutes)
# Run this the MOMENT you get paged

SVC=${1:-"unknown"}
NS=${2:-"production"}
START_TIME=$(date -u)

echo "╔═══════════════════════════════════════════════════════╗"
echo "║  P0 INCIDENT RESPONSE — $(date -u)     ║"
echo "║  Service: $SVC | Namespace: $NS                       ║"
echo "╚═══════════════════════════════════════════════════════╝"
echo ""
echo "Incident Commander: $(whoami)"
echo "Communication Channel: #incident-$(date +%Y%m%d%H%M)"
echo ""

# ── MINUTE 0-1: Immediate triage ──────────────────────────
echo "═══ MINUTE 0-1: Triage ═══"

echo "▶ Service endpoints:"
kubectl get endpoints -n "$NS" "$SVC" -o wide

echo ""
echo "▶ Pod status:"
kubectl get pods -n "$NS" -l "app=$SVC" \
  -o custom-columns='NAME:.metadata.name,STATUS:.status.phase,READY:.status.containerStatuses[0].ready,RESTARTS:.status.containerStatuses[0].restartCount,NODE:.spec.nodeName'

echo ""
echo "▶ Recent events (WARNING):"
kubectl get events -n "$NS" --field-selector type=Warning \
  --sort-by='.lastTimestamp' | tail -20

# ── MINUTE 1-2: Identify failure mode ─────────────────────
echo ""
echo "═══ MINUTE 1-2: Failure Mode ═══"

# CrashLoopBackOff?
CRASHLOOP=$(kubectl get pods -n "$NS" -l "app=$SVC" -o json | \
  jq -r '.items[] | select(.status.containerStatuses[0].state.waiting.reason=="CrashLoopBackOff") | .metadata.name')
if [ -n "$CRASHLOOP" ]; then
  echo "🔴 CRASHLOOPBACKOFF detected: $CRASHLOOP"
  echo "Getting crash logs..."
  kubectl logs -n "$NS" "$CRASHLOOP" --previous --tail=50
fi

# OOMKilled?
OOM=$(kubectl get pods -n "$NS" -l "app=$SVC" -o json | \
  jq -r '.items[] | select(.status.containerStatuses[0].lastState.terminated.reason=="OOMKilled") | .metadata.name')
if [ -n "$OOM" ]; then
  echo "🔴 OOMKilled detected: $OOM"
  echo "Current memory limits:"
  kubectl get pod "$OOM" -n "$NS" -o jsonpath='{.spec.containers[0].resources}'
fi

# ImagePullBackOff?
IMG=$(kubectl get pods -n "$NS" -l "app=$SVC" -o json | \
  jq -r '.items[] | select(.status.containerStatuses[0].state.waiting.reason=="ImagePullBackOff") | .metadata.name')
if [ -n "$IMG" ]; then
  echo "🔴 ImagePullBackOff: $IMG"
  kubectl describe pod "$IMG" -n "$NS" | grep -A 5 "Events:"
fi

# ── MINUTE 2-3: Check recent deploys ──────────────────────
echo ""
echo "═══ MINUTE 2-3: Recent Changes ═══"
echo "Last 3 Helm releases:"
helm history "$SVC" -n "$NS" --max 3 2>/dev/null || echo "No Helm release found"

echo ""
echo "Last 3 deployment revisions:"
kubectl rollout history deployment/"$SVC" -n "$NS" 2>/dev/null || true

# ── MINUTE 3-5: Immediate mitigation options ──────────────
echo ""
echo "═══ MITIGATION OPTIONS ═══"
echo ""
echo "Option A — Rollback (fastest, needs HITL approval):"
echo "  kubectl rollout undo deployment/$SVC -n $NS"
echo "  helm rollback $SVC -n $NS"
echo ""
echo "Option B — Scale up (if capacity issue):"
echo "  kubectl scale deployment/$SVC -n $NS --replicas=10"
echo ""
echo "Option C — Restart pods (if hanging):"
echo "  kubectl rollout restart deployment/$SVC -n $NS"
echo ""
echo "[SELECT OPTION AND GET APPROVAL BEFORE EXECUTING]"
echo "═══════════════════════════════════════════════════"
```

---

### Playbook-002: Memory Leak Debug

```bash
#!/bin/bash
# memory-leak.sh — Diagnose memory growth in production

NS=$1
SVC=$2

echo "══ MEMORY LEAK INVESTIGATION ══"
echo "Service: $SVC | Namespace: $NS"

# Current memory usage
echo ""
echo "▶ Current memory usage:"
kubectl top pods -n "$NS" -l "app=$SVC" --sort-by=memory

# Memory limits vs requests
echo ""
echo "▶ Memory config:"
kubectl get pods -n "$NS" -l "app=$SVC" -o json | \
  jq -r '.items[].spec.containers[] | "\(.name): requests=\(.resources.requests.memory // "none") limits=\(.resources.limits.memory // "UNLIMITED 🚨")"'

# OOM events in last hour
echo ""
echo "▶ OOM events (last 1h):"
kubectl get events -n "$NS" \
  --field-selector reason=OOMKilling \
  --sort-by='.lastTimestamp' | \
  awk -v cutoff="$(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M')" '$0 > cutoff'

# Heap dump (Node.js)
echo ""
echo "▶ Heap snapshot (if Node.js):"
POD=$(kubectl get pod -n "$NS" -l "app=$SVC" -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n "$NS" "$POD" -- \
  kill -USR2 1 2>/dev/null && \
  echo "Heap dump triggered — check app logs for file path" || \
  echo "Not a Node.js app or USR2 not handled"

# Prometheus memory trend (last 2h)
echo ""
echo "▶ Prometheus query for memory trend:"
echo "  container_memory_working_set_bytes{namespace='$NS',pod=~'$SVC-.*'}"
echo "  → Check Grafana dashboard: Memory Usage panel"

# Recommendations
echo ""
echo "══ RECOMMENDATIONS ══"
echo "1. If memory grows unbounded → likely memory leak"
echo "   → Profile: kubectl exec $POD -- node --inspect"
echo "2. If memory hits limit → increase limit or optimize"
echo "   → Patch: kubectl set resources deployment/$SVC -n $NS --limits=memory=2Gi"
echo "3. If periodic spikes → likely batch job or GC issue"
echo "   → Add memory pressure handling in app code"
```

---

### Playbook-003: Database Connection Pool Exhaustion

```bash
#!/bin/bash
# db-connpool.sh — Debug DB connection exhaustion

NS=$1
APP=$2

echo "══ DB CONNECTION POOL DEBUG ══"

# App-side: current connection count (via app metrics)
echo ""
echo "▶ Application DB pool metrics:"
kubectl exec -n "$NS" $(kubectl get pod -n "$NS" -l "app=$APP" -o name | head -1) -- \
  wget -qO- http://localhost:9090/metrics 2>/dev/null | \
  grep -E "db_pool|connection" || echo "  No pool metrics exposed"

# DB side: active connections (PostgreSQL)
echo ""
echo "▶ PostgreSQL active connections:"
kubectl exec -n "$NS" $(kubectl get pod -n "$NS" -l "app=postgres" -o name | head -1) -- \
  psql -U postgres -c "
    SELECT 
      count(*) as total,
      count(*) FILTER (WHERE state = 'active') as active,
      count(*) FILTER (WHERE state = 'idle') as idle,
      count(*) FILTER (WHERE state = 'idle in transaction') as idle_in_tx,
      max_conn
    FROM pg_stat_activity
    CROSS JOIN (SELECT setting::int as max_conn FROM pg_settings WHERE name='max_connections') m;
  " 2>/dev/null || echo "  Cannot connect to Postgres directly"

# Long-running queries
echo ""
echo "▶ Long-running queries (> 30s):"
kubectl exec -n "$NS" postgres-0 -- \
  psql -U postgres -c "
    SELECT pid, now() - query_start AS duration, state, query 
    FROM pg_stat_activity 
    WHERE query_start < now() - interval '30 seconds'
    AND state != 'idle'
    ORDER BY duration DESC LIMIT 10;
  " 2>/dev/null || true

echo ""
echo "══ MITIGATION ══"
echo "1. Kill idle-in-transaction connections:"
echo "   SELECT pg_terminate_backend(pid) FROM pg_stat_activity"
echo "   WHERE state = 'idle in transaction' AND duration > interval '5 min';"
echo ""
echo "2. Restart PgBouncer (connection pooler) if deployed"
echo "3. Scale app replicas DOWN to reduce connection count"
echo "4. Increase max_connections (requires DB restart - HITL required)"
```

---

## HITL Incident Escalation Matrix

```
┌──────────────────────────────────────────────────────────────────┐
│                   INCIDENT SEVERITY & APPROVAL MATRIX            │
├──────────┬────────────────────┬─────────────┬────────────────────┤
│ Severity │ Definition          │ Response    │ Approval Required  │
├──────────┼────────────────────┼─────────────┼────────────────────┤
│    P0    │ Full outage         │ < 5 min     │ 2 engineers + mgmt │
│          │ >50% error rate     │             │ before ANY action  │
├──────────┼────────────────────┼─────────────┼────────────────────┤
│    P1    │ Partial outage      │ < 15 min    │ 1 senior engineer  │
│          │ 10-50% error rate   │             │ for prod changes   │
├──────────┼────────────────────┼─────────────┼────────────────────┤
│    P2    │ Degraded perf       │ < 1 hour    │ Team lead approval │
│          │ <10% error rate     │             │ for prod changes   │
├──────────┼────────────────────┼─────────────┼────────────────────┤
│    P3    │ Minor issue         │ < 1 day     │ Self-service with  │
│          │ No user impact      │             │ PR review          │
└──────────┴────────────────────┴─────────────┴────────────────────┘

ESCALATION PATH:
On-call SRE → Team Lead → Platform Director → CTO

AUTO-PAGE TRIGGERS (PagerDuty):
  • Error rate > 1% for 2+ min      → P1
  • Error rate > 10% for 1+ min     → P0
  • Latency P99 > 5s for 3+ min     → P1
  • All pods in a deployment down   → P0
  • Cluster nodes not ready         → P0
  • DB connections > 90% max        → P1
  • Cert expiry < 7 days            → P2
```

---

## Post-Incident Review (PIR) Template

```markdown
## Post-Incident Review — [Service] — [Date]

**Severity:** P[0-3]
**Duration:** [start] → [end] ([total minutes])
**Impact:** [users affected, revenue impact, SLA breach?]

### Timeline
| Time | Event | Person |
|------|-------|--------|
| HH:MM | Monitoring alert fired | PagerDuty |
| HH:MM | On-call acknowledged | @engineer |
| HH:MM | Root cause identified | @engineer |
| HH:MM | Fix deployed | @engineer |
| HH:MM | Service restored | System |
| HH:MM | All-clear | @engineer |

### Root Cause
[One paragraph — what actually failed and why]

### Contributing Factors
- [ ] Missing monitoring/alerting
- [ ] Untested code path
- [ ] Configuration drift
- [ ] Insufficient testing in staging
- [ ] Missing runbook

### Action Items
| Priority | Action | Owner | Due Date |
|----------|--------|-------|----------|
| P0 | Fix root cause | @engineer | [date] |
| P1 | Add test coverage | @engineer | [date] |
| P2 | Update runbook | @engineer | [date] |

### What Went Well
- [List things that worked]

### What Needs Improvement
- [List gaps identified]
```

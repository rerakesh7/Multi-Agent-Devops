# ⎈ Agent-007: Helm Production Agent + Failures Debugger
# Chart testing · Internal sharing · Production-safe upgrades
# HITL: ALWAYS required for helm upgrade in prod

```
╔══════════════════════════════════════════════════════════════════╗
║           HELM PRODUCTION AGENT v2.0                             ║
║   Failures Debug · Chart Testing · OCI Registry Sharing         ║
║   ⚠️  ALL HELM UPGRADES IN PROD REQUIRE HUMAN APPROVAL            ║
╚══════════════════════════════════════════════════════════════════╝

HELM ARCHITECTURE:

  ┌─────────────────────────────────────────────────────────────┐
  │                 INTERNAL CHART REGISTRY (OCI)                │
  │                                                              │
  │   charts/                                                    │
  │   ├── base-service/          (parent chart)                 │
  │   │   ├── Chart.yaml                                        │
  │   │   ├── values.yaml        (sensible defaults)            │
  │   │   └── templates/                                        │
  │   │       ├── deployment.yaml                               │
  │   │       ├── service.yaml                                  │
  │   │       ├── hpa.yaml                                      │
  │   │       ├── pdb.yaml                                      │
  │   │       └── _helpers.tpl                                  │
  │   ├── api-service/           (inherits base-service)        │
  │   └── worker-service/        (inherits base-service)        │
  │                                                              │
  │   Registry: oci://ghcr.io/my-org/helm-charts                │
  └─────────────────────────────────────────────────────────────┘
```

---

## Skill: Production-Safe Helm Upgrade

```bash
#!/bin/bash
# helm-deploy.sh — Production-safe Helm upgrade with HITL

set -euo pipefail

CHART=${1:?}
NAMESPACE=${2:?}
ENVIRONMENT=${3:?}
VERSION=${4:?}
VALUES_DIR="helm/values"

echo "══════════════════════════════════════════"
echo " Helm Deploy: $CHART → $ENVIRONMENT"
echo " Version: $VERSION | Namespace: $NAMESPACE"
echo "══════════════════════════════════════════"

# ── Step 1: Pre-upgrade checks ──────────────────────────────
echo ""
echo "▶ [1/5] Pre-upgrade validations"

# Check chart exists
helm show chart "oci://ghcr.io/my-org/helm-charts/$CHART:$VERSION" > /dev/null || {
  echo "❌ Chart version $VERSION not found"
  exit 1
}

# Diff current vs proposed (helm diff plugin)
echo ""
echo "▶ [2/5] Helm diff (what will change)"
helm diff upgrade "$CHART" \
  "oci://ghcr.io/my-org/helm-charts/$CHART:$VERSION" \
  --namespace "$NAMESPACE" \
  -f "${VALUES_DIR}/${ENVIRONMENT}.yaml" \
  --three-way-merge \
  --normalize-manifests \
  --color 2>&1 | head -200

# ── HUMAN APPROVAL GATE ─────────────────────────────────────
if [ "$ENVIRONMENT" = "prod" ] || [ "$ENVIRONMENT" = "production" ]; then
  echo ""
  echo "╔══════════════════════════════════════════════════════╗"
  echo "║  ⚠️  PRODUCTION HELM UPGRADE — APPROVAL REQUIRED      ║"
  echo "║                                                        ║"
  echo "║  Chart:     $CHART                                    ║"
  echo "║  Version:   $VERSION                                  ║"
  echo "║  Namespace: $NAMESPACE                                ║"
  echo "║                                                        ║"
  echo "║  Current:   $(helm list -n $NAMESPACE --filter $CHART -o json | jq -r '.[0].chart')  ║"
  echo "║  Rollback:  helm rollback $CHART -n $NAMESPACE        ║"
  echo "╚══════════════════════════════════════════════════════╝"
  read -p "Type 'APPROVE' to proceed: " CONFIRM
  [ "$CONFIRM" = "APPROVE" ] || { echo "❌ Cancelled"; exit 0; }
fi
# ────────────────────────────────────────────────────────────

# ── Step 2: Backup current release ────────────────────────
echo ""
echo "▶ [3/5] Saving current release manifest"
helm get manifest "$CHART" -n "$NAMESPACE" > "/tmp/${CHART}-backup-$(date +%s).yaml" 2>/dev/null || true
CURRENT_REVISION=$(helm list -n "$NAMESPACE" --filter "$CHART" -o json | jq '.[0].revision // 0')
echo "Current revision: $CURRENT_REVISION"

# ── Step 3: Upgrade with atomic flag ─────────────────────
echo ""
echo "▶ [4/5] Running helm upgrade"
helm upgrade "$CHART" \
  "oci://ghcr.io/my-org/helm-charts/$CHART:$VERSION" \
  --namespace "$NAMESPACE" \
  --create-namespace \
  -f "${VALUES_DIR}/common.yaml" \
  -f "${VALUES_DIR}/${ENVIRONMENT}.yaml" \
  --atomic \           # Rollback automatically if hooks/pods fail
  --timeout 10m \
  --cleanup-on-fail \
  --history-max 10 \   # Keep last 10 releases for rollback
  --wait \
  --wait-for-jobs \
  --set "image.tag=$VERSION" \
  --set "deploy.timestamp=$(date -u +%Y-%m-%dT%H:%M:%SZ)"

# ── Step 4: Post-upgrade validation ───────────────────────
echo ""
echo "▶ [5/5] Post-upgrade health check"
sleep 15
READY=$(kubectl get deployment -n "$NAMESPACE" -l "app.kubernetes.io/name=$CHART" \
  -o jsonpath='{.items[0].status.readyReplicas}')
DESIRED=$(kubectl get deployment -n "$NAMESPACE" -l "app.kubernetes.io/name=$CHART" \
  -o jsonpath='{.items[0].spec.replicas}')

if [ "$READY" != "$DESIRED" ]; then
  echo "❌ Not all replicas ready ($READY/$DESIRED). Consider rollback:"
  echo "   helm rollback $CHART -n $NAMESPACE"
  exit 1
fi

echo "✅ Upgrade successful: $CHART $VERSION in $NAMESPACE ($READY/$DESIRED pods ready)"
```

---

## Skill: Helm Failures Debug Playbook

### Failure Matrix
```
┌──────────────────────────────────────────────────────────────┐
│              HELM FAILURE DIAGNOSIS MATRIX                    │
├────────────────────────┬─────────────────────────────────────┤
│ Error Message           │ Root Cause & Fix                   │
├────────────────────────┼─────────────────────────────────────┤
│ "timed out waiting for  │ Pods not starting. Check:          │
│  condition"             │ kubectl get events -n <ns>         │
│                         │ kubectl describe pod <pod>         │
├────────────────────────┼─────────────────────────────────────┤
│ "rendered manifests     │ Template error or invalid values   │
│  contain a resource     │ Run: helm template . -f vals.yaml  │
│  that already exists"   │ Fix: --force flag or delete res    │
├────────────────────────┼─────────────────────────────────────┤
│ "release: not found"    │ First install vs upgrade confusion │
│                         │ Use: helm upgrade --install        │
├────────────────────────┼─────────────────────────────────────┤
│ "UPGRADE FAILED:        │ Kubernetes API deprecated          │
│  unable to build        │ Check: pluto detect-helm-releases  │
│  kubernetes objects"    │ Fix: Update apiVersion in chart    │
├────────────────────────┼─────────────────────────────────────┤
│ "post-install hook      │ Hook pod/job failed                │
│  timed out"             │ kubectl get jobs -n <ns>           │
│                         │ kubectl logs job/<hook-job>        │
├────────────────────────┼─────────────────────────────────────┤
│ "another operation      │ Previous release in pending state  │
│ (install/upgrade/       │ Fix: helm rollback or delete       │
│ rollback) is in         │ release secret:                    │
│ progress"               │ kubectl delete secret              │
│                         │   sh.helm.release.v1.<name>.*     │
│                         │   -n <ns> --field-selector         │
│                         │   status=pending-upgrade           │
└────────────────────────┴─────────────────────────────────────┘
```

### Debug Script
```bash
#!/bin/bash
# helm-debug.sh — Comprehensive Helm failure analysis

RELEASE=${1:?}
NAMESPACE=${2:?}

echo "═══════════════════════════════════════"
echo " Helm Debug: $RELEASE ($NAMESPACE)"
echo "═══════════════════════════════════════"

# Release history
echo ""
echo "▶ Release History"
helm history "$RELEASE" -n "$NAMESPACE" --max 10

# Current status
echo ""
echo "▶ Release Status"
helm status "$RELEASE" -n "$NAMESPACE"

# Get rendered manifests from last failed release
echo ""
echo "▶ Failed Release Values"
helm get values "$RELEASE" -n "$NAMESPACE" --all

# Check pods
echo ""
echo "▶ Pod Status"
kubectl get pods -n "$NAMESPACE" -l "app.kubernetes.io/instance=$RELEASE" \
  --sort-by='.status.startTime'

# Events (last 10 min, warnings only)
echo ""
echo "▶ Recent Warning Events"
kubectl get events -n "$NAMESPACE" \
  --field-selector reason!=Scheduled,reason!=Started,reason!=Pulled \
  --sort-by='.lastTimestamp' | tail -30

# Detect stuck pending releases
echo ""
echo "▶ Pending Release Secrets"
kubectl get secrets -n "$NAMESPACE" \
  -l owner=helm \
  -o jsonpath='{range .items[*]}{.metadata.name}: {.metadata.labels.status}{"\n"}{end}' | \
  grep -v "deployed"

# Deprecated API check
echo ""
echo "▶ Deprecated API Versions"
helm template "$RELEASE" -n "$NAMESPACE" 2>/dev/null | \
  grep "apiVersion:" | sort -u | \
  while read line; do
    API=$(echo $line | awk '{print $2}')
    case $API in
      extensions/v1beta1|apps/v1beta1|apps/v1beta2)
        echo "  ⚠️  DEPRECATED: $API"
        ;;
    esac
  done

echo ""
echo "═══ DEBUG COMPLETE ==="
echo "Next steps:"
echo "  1. helm rollback $RELEASE -n $NAMESPACE  (immediate relief)"
echo "  2. Fix values/templates based on above"
echo "  3. helm upgrade --dry-run before next attempt"
```

---

## Skill: Internal Chart Library (Base Service)

### `charts/base-service/Chart.yaml`
```yaml
apiVersion: v2
name: base-service
description: Base chart for all internal microservices
type: library
version: 2.4.0
home: https://github.com/my-org/helm-charts
maintainers:
- name: Platform Team
  email: platform@mycompany.com
annotations:
  category: base
```

### `charts/base-service/values.yaml`
```yaml
# Opinionated defaults — all production-safe
replicaCount: 2             # Minimum for HA

image:
  repository: ""
  tag: ""
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

podDisruptionBudget:
  enabled: true
  minAvailable: "50%"

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3

livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 15
  failureThreshold: 3

securityContext:
  runAsNonRoot: true
  runAsUser: 1001
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]

podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "9090"
  prometheus.io/path: "/metrics"

topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: "{{ .Release.Name }}"
```

---

## Skill: Chart Testing (ct + helm unittest)

```yaml
# .github/workflows/chart-test.yml
name: Chart Testing
on:
  pull_request:
    paths: ['charts/**']

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
      with:
        fetch-depth: 0

    - name: Set up chart-testing
      uses: helm/chart-testing-action@e6669bcd63d7cb57cb77fb9d3955b8da2d3e7af2  # v2.6.1

    - name: Run chart-testing lint
      run: ct lint --target-branch ${{ github.event.repository.default_branch }}

    - name: Create KinD cluster
      uses: helm/kind-action@0025e74a8c7512023d06dc019c617aa3cf561fde  # v1.10.0
      if: steps.list-changed.outputs.changed == 'true'

    - name: Install chart-testing
      run: ct install --target-branch ${{ github.event.repository.default_branch }}

    - name: Helm unittest
      run: |
        helm plugin install https://github.com/helm-unittest/helm-unittest
        for chart in $(find charts -name Chart.yaml -not -path "*/charts/*" -exec dirname {} \;); do
          echo "Testing $chart..."
          helm unittest "$chart" -f 'tests/**/*_test.yaml'
        done

    - name: Pluto deprecated API check
      run: |
        curl -sL https://github.com/FairwindsOps/pluto/releases/latest/download/pluto_linux_amd64.tar.gz | tar xz
        for chart in $(find charts -name Chart.yaml -not -path "*/charts/*" -exec dirname {} \;); do
          helm template test-release "$chart" | ./pluto detect - || true
        done
```

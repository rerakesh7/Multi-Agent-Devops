# 🐳 Agent-005: Docker & Containers Agent
# Multi-cloud container strategy · Image cleanup · Multi-stage Dockerfile

```
╔══════════════════════════════════════════════════════════════════╗
║           DOCKER & CONTAINERS AGENT v1.0                         ║
║   Multi-Cloud Strategy · Security · Image Lifecycle              ║
╚══════════════════════════════════════════════════════════════════╝

MULTI-CLOUD CONTAINER REGISTRY STRATEGY:

  ┌─────────────────────────────────────────────────────────┐
  │                 CONTAINER REGISTRY TOPOLOGY              │
  │                                                          │
  │  CI Build ──▶ GHCR (build cache + staging artifacts)    │
  │                  │                                       │
  │         ┌────────┴─────────┐                           │
  │         ▼                  ▼                           │
  │      GCR/AR              ECR                           │
  │    (GKE prod)          (EKS prod)                      │
  │         │                  │                           │
  │         └────────┬─────────┘                           │
  │                  ▼                                      │
  │                ACR                                      │
  │            (AKS prod)                                   │
  │                                                          │
  │  Replication: GHCR → GCR/ECR/ACR via GitHub Actions     │
  └─────────────────────────────────────────────────────────┘
```

---

## Skill: Multi-Stage Dockerfile (Production Grade)

### Node.js Application
```dockerfile
# syntax=docker/dockerfile:1.7
# Multi-stage: deps → test → build → production
# Result: 25MB final image (vs 800MB single-stage)

#──────────────────────────────────────────────────────────
# Stage 1: Base — common dependencies
#──────────────────────────────────────────────────────────
FROM node:20.18-alpine3.20 AS base
WORKDIR /app

# Install OS deps (pin versions)
RUN apk add --no-cache \
    dumb-init=1.2.5-r3 \    
    && rm -rf /var/cache/apk/*

COPY package*.json ./

#──────────────────────────────────────────────────────────
# Stage 2: Dependencies (dev + prod)
#──────────────────────────────────────────────────────────
FROM base AS deps
RUN --mount=type=cache,target=/root/.npm \
    npm ci --include=dev

#──────────────────────────────────────────────────────────
# Stage 3: Test (run in CI, not in final image)
#──────────────────────────────────────────────────────────
FROM deps AS test
COPY . .
RUN npm run lint && npm run test:coverage
# If tests fail → build fails → nothing pushed to registry

#──────────────────────────────────────────────────────────
# Stage 4: Build (compile TypeScript/Next.js/etc.)
#──────────────────────────────────────────────────────────
FROM deps AS builder
COPY . .
RUN npm run build && \
    npm prune --production

#──────────────────────────────────────────────────────────
# Stage 5: Production (minimal, non-root, read-only FS)
#──────────────────────────────────────────────────────────
FROM node:20.18-alpine3.20 AS production

# Security: Create non-root user
RUN addgroup -g 1001 appgroup && \
    adduser -D -u 1001 -G appgroup appuser

WORKDIR /app

# Copy only production artifacts
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:appgroup /app/package.json .
COPY --from=base /usr/bin/dumb-init /usr/bin/dumb-init

# Security hardening
USER appuser
ENV NODE_ENV=production \
    PORT=8080

# Read-only filesystem (uncomment if your app supports it)
# RUN chmod -R 555 /app

EXPOSE 8080

# Health check baked into image
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD wget -qO- http://localhost:8080/health || exit 1

# Use dumb-init as PID 1 (proper signal handling)
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/server.js"]
```

### Python Application (FastAPI/Django)
```dockerfile
# syntax=docker/dockerfile:1.7
FROM python:3.12-slim-bookworm AS base
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

#──────────────────────────────────────────────────────────
FROM base AS builder
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential=12.9 \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --prefix=/install -r requirements.txt

#──────────────────────────────────────────────────────────
FROM base AS production
COPY --from=builder /install /usr/local

RUN adduser --disabled-password --gecos "" --uid 1001 appuser
USER appuser
WORKDIR /app
COPY --chown=appuser:appuser . .

HEALTHCHECK --interval=30s --timeout=10s --start-period=20s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"

EXPOSE 8000
CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Go Application (Scratch-based — 8MB!)
```dockerfile
# syntax=docker/dockerfile:1.7
FROM golang:1.23-alpine3.20 AS builder
WORKDIR /build
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-w -s -X main.version=${VERSION}" \
    -o /app ./cmd/server

# Verify binary
RUN /app --version

#──────────────────────────────────────────────────────────
FROM scratch AS production
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app /app
USER 65534:65534    # nobody
EXPOSE 8080
HEALTHCHECK --interval=30s CMD ["/app", "healthcheck"]
ENTRYPOINT ["/app"]
```

---

## Skill: Image Cleanup & Monitoring

### Multi-Cloud Image Retention Policy

```bash
#!/bin/bash
# image-cleanup.sh — Run weekly via GitHub Actions scheduled workflow
# HITL: Dry-run first, human approves deletion

set -euo pipefail

# Configuration
DAYS_TO_KEEP=${DAYS_TO_KEEP:-30}
KEEP_TAGGED=${KEEP_TAGGED:-true}   # Always keep tagged images
DRY_RUN=${DRY_RUN:-true}          # Default: dry run

echo "═══════════════════════════════════════"
echo " Image Cleanup — $(date -u)"
echo " Days to keep: $DAYS_TO_KEEP"
echo " Dry run: $DRY_RUN"
echo "═══════════════════════════════════════"

# ── GCP Artifact Registry ────────────────────────────────
cleanup_gcr() {
  local REPO=$1
  echo ""
  echo "▶ GCR/Artifact Registry: $REPO"
  
  gcloud artifacts docker images list "$REPO" \
    --filter="updateTime < -P${DAYS_TO_KEEP}D AND NOT tags:*" \
    --format="value(IMAGE, DIGEST)" | \
  while read IMAGE DIGEST; do
    echo "  Would delete: $IMAGE@$DIGEST"
    if [ "$DRY_RUN" = "false" ]; then
      gcloud artifacts docker images delete "${IMAGE}@${DIGEST}" --quiet
      echo "  ✅ Deleted"
    fi
  done
}

# ── AWS ECR ──────────────────────────────────────────────
cleanup_ecr() {
  local REPO=$1
  echo ""
  echo "▶ ECR: $REPO"
  
  CUTOFF=$(date -u -d "${DAYS_TO_KEEP} days ago" +%s)
  
  aws ecr describe-images \
    --repository-name "$REPO" \
    --query "imageDetails[?imagePushedAt<\`$(date -u -d "${DAYS_TO_KEEP} days ago" +%Y-%m-%dT%H:%M:%S)\`] | [?!imageTagsCount]" \
    --output json | \
  jq -r '.[].imageDigest' | \
  while read DIGEST; do
    echo "  Would delete: $DIGEST"
    if [ "$DRY_RUN" = "false" ]; then
      aws ecr batch-delete-image \
        --repository-name "$REPO" \
        --image-ids imageDigest=$DIGEST
    fi
  done
}

# ── Azure ACR ────────────────────────────────────────────
cleanup_acr() {
  local REGISTRY=$1
  local REPO=$2
  echo ""
  echo "▶ ACR: $REGISTRY/$REPO"
  
  az acr manifest list-metadata \
    --registry "$REGISTRY" \
    --name "$REPO" \
    --query "[?lastUpdateTime < '$(date -u -d "${DAYS_TO_KEEP} days ago" +%Y-%m-%dT%H:%M:%SZ)' && tags == null]" \
    --output tsv | awk '{print $1}' | \
  while read DIGEST; do
    echo "  Would delete: $DIGEST"
    if [ "$DRY_RUN" = "false" ]; then
      az acr repository delete \
        --name "$REGISTRY" \
        --image "${REPO}@${DIGEST}" --yes
    fi
  done
}

# Run cleanup
cleanup_gcr "us-central1-docker.pkg.dev/my-project/production"
cleanup_ecr "production/api"
cleanup_acr "myregistry" "api"

echo ""
if [ "$DRY_RUN" = "true" ]; then
  echo "═══ DRY RUN COMPLETE — No images deleted ==="
  echo "Re-run with DRY_RUN=false after human review"
else
  echo "═══ CLEANUP COMPLETE ==="
fi
```

### Image Size Monitoring
```yaml
# .github/workflows/image-analysis.yml
name: Image Analysis
on:
  push:
    branches: [main]

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
    - name: Dive image analysis
      run: |
        docker pull wagoodman/dive:latest
        docker run --rm \
          -v /var/run/docker.sock:/var/run/docker.sock \
          wagoodman/dive:latest \
          --ci --lowestEfficiency=0.9 \
          ${{ env.IMAGE_TAG }}

    - name: Size regression check
      run: |
        NEW_SIZE=$(docker inspect --format='{{.Size}}' $IMAGE_TAG)
        BASELINE=$(cat .image-size-baseline 2>/dev/null || echo 0)
        DIFF=$((NEW_SIZE - BASELINE))
        THRESHOLD=$((50 * 1024 * 1024))  # 50MB threshold
        
        if [ $DIFF -gt $THRESHOLD ]; then
          echo "❌ Image grew by $(($DIFF / 1024 / 1024))MB — exceeds 50MB threshold"
          exit 1
        fi
        echo "✅ Image size OK: $(($NEW_SIZE / 1024 / 1024))MB"
```

---

## Skill: Multi-Cloud Container Strategy

```
┌─────────────────────────────────────────────────────────────┐
│           CONTAINER STRATEGY BY CLOUD                        │
├──────────────┬──────────────┬───────────────────────────────┤
│    GCP/GKE   │   AWS/EKS    │       Azure/AKS               │
├──────────────┼──────────────┼───────────────────────────────┤
│ Artifact Reg │ ECR          │ ACR (with geo-replication)    │
│ Binary Auth  │ ECR signing  │ ACR Tasks (auto-build)        │
│ Container    │ Inspector    │ Defender for Containers       │
│ Analysis API │ Snyk/Trivy   │ Microsoft Defender            │
│ cos_containerd│ Bottlerocket│ CBL-Mariner node OS           │
│ Workload ID  │ IRSA         │ Azure AD Workload Identity    │
└──────────────┴──────────────┴───────────────────────────────┘
```

### Security Scanning Pipeline
```yaml
# Run before ANY image push to production registry
- name: Trivy scan (block on CRITICAL)
  run: |
    trivy image \
      --exit-code 1 \
      --severity CRITICAL \
      --format sarif \
      --output trivy-results.sarif \
      $IMAGE_TAG
      
- name: Grype SBOM scan
  run: |
    grype $IMAGE_TAG \
      --fail-on critical \
      --output table

- name: Cosign sign (after scan passes)
  run: |
    cosign sign \
      --key env://COSIGN_PRIVATE_KEY \
      $IMAGE_TAG@${{ steps.push.outputs.digest }}
```

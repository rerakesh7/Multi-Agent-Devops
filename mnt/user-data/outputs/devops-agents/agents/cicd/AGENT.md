# ⚙️ Agent-004: CI/CD & GitHub Actions Agent
# Covers: Reusable Workflows · Concurrency · Bad Config Prevention · Deployment Safety

```
╔══════════════════════════════════════════════════════════════════╗
║           CI/CD & GITHUB ACTIONS AGENT v2.0                      ║
║   Config Guardrails · Reusable Workflows · Safe Deploys          ║
╚══════════════════════════════════════════════════════════════════╝

PIPELINE ARCHITECTURE:

  PR Open          Merge to Main        Manual Trigger
     │                   │                    │
     ▼                   ▼                    ▼
  ┌──────┐          ┌─────────┐          ┌─────────┐
  │  CI  │          │  Build  │          │ Promote │
  │      │          │  +Push  │          │  to     │
  │ lint │          │  Image  │          │  Prod   │
  │ test │          └────┬────┘          └────┬────┘
  │ scan │               │                    │
  │ val  │          ┌────▼────┐               │
  └──────┘          │ Deploy  │    [HUMAN]────▶│
                    │ Staging │    APPROVAL    │
                    └─────────┘               │
                                         ┌────▼────┐
                                         │ Deploy  │
                                         │  Prod   │
                                         └─────────┘
```

---

## Skill: Reusable Workflow Library

### `.github/workflows/reusable-ci.yml`
```yaml
name: Reusable CI
on:
  workflow_call:
    inputs:
      node-version:
        type: string
        default: '20'
      run-e2e:
        type: boolean
        default: false
      image-name:
        type: string
        required: true
    secrets:
      registry-token:
        required: true
      sonar-token:
        required: false

jobs:
  lint-and-test:
    name: Lint, Test & Scan
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    steps:
    - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2

    - name: Setup Node.js
      uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af  # v4.1.0
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'

    - name: Install dependencies
      run: npm ci --prefer-offline

    - name: Lint
      run: npm run lint

    - name: Unit tests with coverage
      run: npm run test:coverage

    - name: Upload coverage
      uses: codecov/codecov-action@e28ff129e5465c2c0dcc6f003fc735cb6ae0c673  # v4.5.0

    - name: Trivy vulnerability scan
      uses: aquasecurity/trivy-action@6e7b7d1fd3e4fef0c5fa8cce1229c54b2c9bd0d8  # v0.24.0
      with:
        scan-type: 'fs'
        severity: 'CRITICAL,HIGH'
        exit-code: '1'

  build-image:
    name: Build & Push Image
    needs: lint-and-test
    runs-on: ubuntu-latest
    outputs:
      image-digest: ${{ steps.push.outputs.digest }}
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
    - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683

    - name: Docker meta
      id: meta
      uses: docker/metadata-action@902fa8ec7d6ecbea8a64ac4a9c8dc16fc4d8a4fb  # v5.6.1
      with:
        images: ${{ inputs.image-name }}
        tags: |
          type=sha,prefix={{branch}}-
          type=semver,pattern={{version}}
          type=ref,event=pr

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@b5ca514318bd6ebac0fb2aedd5d36ec1b5c232a2  # v3.10.0

    - name: Login to Registry
      uses: docker/login-action@9780b0c442fbb1117ed29e0efdff1e18412f7567  # v3.3.0
      with:
        password: ${{ secrets.registry-token }}

    - name: Build and push
      id: push
      uses: docker/build-push-action@471d1dc4e07e5cdedd4c0b47bbf1b5b4588db9e3  # v6.9.0
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
        provenance: true
        sbom: true
```

---

### `.github/workflows/reusable-deploy.yml`
```yaml
name: Reusable Deploy
on:
  workflow_call:
    inputs:
      environment:
        type: string
        required: true       # dev | staging | prod
      cluster:
        type: string
        required: true       # gke-us-central1 | eks-us-east-1 | aks-eastus
      namespace:
        type: string
        required: true
      image-tag:
        type: string
        required: true
      strategy:
        type: string
        default: rolling     # rolling | canary | blue-green
    secrets:
      kubeconfig:
        required: true

jobs:
  validate-config:
    name: Validate Configs (prevent bad configs)
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683

    # ── CONFIG VALIDATION GATE ─────────────────────────────
    - name: Validate Kubernetes manifests
      run: |
        # Install tools
        curl -sL https://raw.githubusercontent.com/yannh/kubeconform/master/scripts/download.sh | sh
        curl -sL https://github.com/open-policy-agent/conftest/releases/latest/download/conftest_Linux_x86_64.tar.gz | tar xz

        # Schema validation
        ./kubeconform -strict -summary -output json k8s/ | tee validation.json
        if jq '.summary.invalid > 0' validation.json | grep -q true; then
          echo "❌ Kubernetes manifest validation failed"
          exit 1
        fi

        # Policy validation (OPA)
        ./conftest test k8s/ --policy policies/ --all-namespaces
        echo "✅ All manifests valid"

    - name: Helm lint
      if: hashFiles('helm/') != ''
      run: |
        helm lint helm/${{ inputs.namespace }}/ \
          -f helm/${{ inputs.namespace }}/values-${{ inputs.environment }}.yaml \
          --strict
        echo "✅ Helm lint passed"

    - name: Detect secret in config
      run: |
        # Detect accidental secrets in manifests
        git diff HEAD~1 -- k8s/ helm/ | \
          grep -E '(password|secret|token|key|credential)\s*[:=]\s*\S{8,}' | \
          grep -v '^\+\+\+\|^---' && echo "❌ Possible secret in config!" && exit 1 || true
        echo "✅ No secrets detected in configs"

  deploy:
    name: Deploy to ${{ inputs.environment }}
    needs: validate-config
    runs-on: ubuntu-latest
    environment:
      name: ${{ inputs.environment }}        # GitHub Environments for protection rules
      url: https://${{ inputs.environment }}.example.com
    concurrency:
      group: deploy-${{ inputs.environment }}-${{ inputs.namespace }}
      cancel-in-progress: false             # NEVER cancel in-progress deploys
    steps:
    - name: Configure kubectl
      run: |
        echo "${{ secrets.kubeconfig }}" | base64 -d > /tmp/kubeconfig
        echo "KUBECONFIG=/tmp/kubeconfig" >> $GITHUB_ENV

    - name: Pre-deploy snapshot
      run: |
        kubectl get all -n ${{ inputs.namespace }} -o yaml > /tmp/pre-deploy-snapshot.yaml
        echo "Pre-deploy snapshot saved"

    - name: Deploy (${{ inputs.strategy }})
      run: |
        case "${{ inputs.strategy }}" in
          canary)
            kubectl argo rollouts set image ${{ inputs.namespace }} \
              app=${{ inputs.image-tag }} -n ${{ inputs.namespace }}
            ;;
          blue-green)
            # Call blue-green script
            bash scripts/bg-switch.sh ${{ inputs.namespace }} ${{ inputs.namespace }} green
            ;;
          rolling|*)
            kubectl set image deployment/${{ inputs.namespace }} \
              app=${{ inputs.image-tag }} -n ${{ inputs.namespace }}
            kubectl rollout status deployment/${{ inputs.namespace }} \
              -n ${{ inputs.namespace }} --timeout=10m
            ;;
        esac

    - name: Post-deploy smoke test
      run: |
        sleep 30
        SVC_URL="https://${{ inputs.environment }}.example.com/health"
        STATUS=$(curl -o /dev/null -s -w "%{http_code}" "$SVC_URL")
        if [ "$STATUS" != "200" ]; then
          echo "❌ Smoke test failed: HTTP $STATUS"
          echo "Rolling back..."
          kubectl rollout undo deployment/${{ inputs.namespace }} -n ${{ inputs.namespace }}
          exit 1
        fi
        echo "✅ Smoke test passed: HTTP $STATUS"

    - name: Upload snapshot on failure
      if: failure()
      uses: actions/upload-artifact@b4b15b8c7c6ac21ea08fcf65892d2ee8f75cf882  # v4.4.3
      with:
        name: pre-deploy-snapshot
        path: /tmp/pre-deploy-snapshot.yaml
```

---

## Skill: Concurrency & Failure Handling

### `.github/workflows/main-pipeline.yml`
```yaml
name: Main Pipeline
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

# ── CONCURRENCY CONTROL ────────────────────────────────────────
# One pipeline per branch; new push cancels old PR runs (not deploys)
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  # ── CHANGE DETECTION (skip unchanged services) ─────────────────
  detect-changes:
    name: Detect Changed Services
    runs-on: ubuntu-latest
    outputs:
      api-changed: ${{ steps.filter.outputs.api }}
      web-changed: ${{ steps.filter.outputs.web }}
      infra-changed: ${{ steps.filter.outputs.infra }}
    steps:
    - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
    - uses: dorny/paths-filter@de90cc6fb38fc0963ad72b210f1f284cd68cea36  # v3.0.2
      id: filter
      with:
        filters: |
          api:
            - 'services/api/**'
          web:
            - 'services/web/**'
          infra:
            - 'terraform/**'
            - '.github/workflows/**'

  # ── CI (CONDITIONAL) ───────────────────────────────────────────
  ci-api:
    needs: detect-changes
    if: needs.detect-changes.outputs.api-changed == 'true'
    uses: ./.github/workflows/reusable-ci.yml
    with:
      image-name: gcr.io/my-project/api
    secrets:
      registry-token: ${{ secrets.GCR_TOKEN }}

  # ── FAILURE HANDLING ───────────────────────────────────────────
  notify-on-failure:
    needs: [ci-api]
    if: failure()
    runs-on: ubuntu-latest
    steps:
    - name: Notify Slack
      uses: slackapi/slack-github-action@37ebaef184d7626c5f204ab8d3baff4262dd30f0  # v2.0.0
      with:
        payload: |
          {
            "text": "❌ Pipeline failed on `${{ github.ref }}` by @${{ github.actor }}",
            "attachments": [{
              "color": "danger",
              "fields": [
                {"title": "Workflow", "value": "${{ github.workflow }}", "short": true},
                {"title": "Run", "value": "${{ github.run_id }}", "short": true},
                {"title": "Commit", "value": "${{ github.sha }}", "short": false}
              ]
            }]
          }
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## Skill: Bad Config Prevention (OPA Policies)

### `policies/k8s-safety.rego`
```rego
package k8s.safety

# DENY: No resource limits set
deny[msg] {
  input.kind == "Deployment"
  container := input.spec.template.spec.containers[_]
  not container.resources.limits
  msg := sprintf("Container '%v' has no resource limits", [container.name])
}

# DENY: Privileged containers
deny[msg] {
  input.kind == "Deployment"
  container := input.spec.template.spec.containers[_]
  container.securityContext.privileged == true
  msg := sprintf("Container '%v' runs as privileged — not allowed", [container.name])
}

# DENY: Latest tag
deny[msg] {
  input.kind == "Deployment"
  container := input.spec.template.spec.containers[_]
  endswith(container.image, ":latest")
  msg := sprintf("Container '%v' uses ':latest' tag — pin to a digest", [container.name])
}

# DENY: No readiness probe
warn[msg] {
  input.kind == "Deployment"
  container := input.spec.template.spec.containers[_]
  not container.readinessProbe
  msg := sprintf("Container '%v' has no readinessProbe", [container.name])
}

# DENY: hostPath volumes (tenant isolation violation)
deny[msg] {
  input.kind == "Deployment"
  volume := input.spec.template.spec.volumes[_]
  volume.hostPath
  msg := sprintf("Volume '%v' uses hostPath — forbidden for security", [volume.name])
}

# DENY: replicas < 2 in prod
deny[msg] {
  input.kind == "Deployment"
  input.metadata.labels["environment"] == "production"
  input.spec.replicas < 2
  msg := "Production deployments must have at least 2 replicas"
}
```

---

## Skill: Deployment Safety Patterns

### Protected Environments Config
```yaml
# GitHub Environments (configure in repo settings)
environments:
  production:
    protection_rules:
      - type: required_reviewers
        reviewers:
          - team: platform-team
        min_reviewers: 1
      - type: wait_timer
        wait_minutes: 5    # 5-min "last chance to abort" window
    deployment_branch_policy:
      protected_branches: true    # Only from main branch
    
  staging:
    protection_rules:
      - type: required_reviewers
        min_reviewers: 0          # Auto-deploy to staging
    deployment_branch_policy:
      protected_branches: false
```

### Deployment Safety Gates Summary
```
GATE 1: PR Checks (automated)
  ✓ lint passes
  ✓ tests pass + coverage > 80%
  ✓ security scan (no CRITICAL CVEs)
  ✓ OPA policy validation
  ✓ no secrets in code

GATE 2: Merge Protection (automated)
  ✓ branch is up to date with main
  ✓ required reviewers approved
  ✓ signed commits (optional)

GATE 3: Staging Deploy (automated)
  ✓ kubeconform validation
  ✓ helm lint
  ✓ deploy to staging
  ✓ smoke test passes
  ✓ Slack notification

GATE 4: Production Deploy (MANUAL)
  ✓ [HUMAN] reviews staging smoke test
  ✓ [HUMAN] approves GitHub Environment
  ✓ 5-minute cool-down window
  ✓ deploy with rollback plan
  ✓ post-deploy monitoring check
```

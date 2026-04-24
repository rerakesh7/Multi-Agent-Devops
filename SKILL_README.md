# DevOps Multi-Agent Orchestration Skill

**Version:** 1.0  
**Author:** Built from production multi-agent system  
**Last Updated:** April 2026

---

## What This Skill Does

A production-grade DevOps orchestration system that provides **8 specialist agents** for multi-cloud infrastructure (GCP/GKE, AWS/EKS, Azure/AKS) with **mandatory Human-in-the-Loop (HITL) approval gates** at every production boundary.

### Coverage

✅ **Multi-tenant Kubernetes isolation** (namespaces, RBAC, NetworkPolicy, ResourceQuota)  
✅ **502/ingress debugging** (NGINX, ALB, App Gateway, upstream timeouts)  
✅ **Zero-downtime deployments** (blue/green, canary, rolling, Argo Rollouts)  
✅ **CI/CD & GitHub Actions** (reusable workflows, OPA bad-config prevention)  
✅ **Docker & containers** (multi-stage Dockerfiles, image lifecycle, multi-cloud registry)  
✅ **Terraform** (multi-env, state management, DB protection, prevent_destroy)  
✅ **Helm** (production-safe upgrades, failure debugging, chart testing)  
✅ **Production incident response** (P0 triage, playbooks, PIR templates)

---

## Installation

### Option 1: Install via Claude.ai/Claude Desktop

1. Download `devops-multi-agent-orchestration.skill`
2. In Claude: Settings → Skills → Install Skill → Select the `.skill` file
3. The skill is now active and will trigger on relevant DevOps queries

### Option 2: Manual Installation (if .skill format not supported)

```bash
# Extract to your skills directory
mkdir -p ~/.claude/skills/devops-multi-agent-orchestration
tar -xzf devops-multi-agent-orchestration.skill -C ~/.claude/skills/devops-multi-agent-orchestration/
```

---

## How It Works

When you ask a DevOps question, the skill:

1. **Routes** your query to the relevant agent(s) using keyword matching
2. **Loads** the appropriate reference file(s) from `references/agents/`
3. **Determines** environment and risk level (dev/staging/prod)
4. **Generates** the solution (YAML, scripts, commands, playbooks)
5. **Gates** any production action behind HITL approval blocks
6. **Provides** rollback plans (< 60 seconds for all prod actions)

### HITL Approval Matrix

```
Environment    Risk       Approval Required
─────────────  ─────────  ───────────────────────────────
dev            any        Auto-approve (logged)
staging        any        Slack notification only
production     HIGH       1 engineer manual approval
production     CRITICAL   2-person rule (two approvals)
```

---

## Example Usage

### Example 1: Debug 502 Errors

**You:** "We're getting 502 errors on our API after deploying to GKE production"

**Skill provides:**
- Immediate 6-step triage script
- Root cause diagnosis matrix
- NGINX annotation fixes
- **HITL approval gate** before applying config changes
- Rollback plan (< 30 seconds)
- Prometheus queries to verify recovery

---

### Example 2: Set Up Canary Deployment

**You:** "Set up canary deploy for payment-service on EKS, we process 50k rpm"

**Skill provides:**
- Decision tree → routes to canary (high traffic)
- Complete Argo Rollouts manifest (5% → 20% → 50% → 100%)
- AnalysisTemplate with Prometheus success-rate metric
- **HITL pause at 20%** (manual promotion required)
- `kubectl argo rollouts promote` and `abort` commands
- PodDisruptionBudget manifest

---

### Example 3: Isolate Tenant on GKE

**You:** "Isolate tenant-42 on GKE, they're premium tier"

**Skill provides:**
- Full Namespace manifest with labels
- ResourceQuota (no NodePorts allowed)
- NetworkPolicy: deny-all + allow-intra-namespace
- RBAC Role and RoleBinding
- GCP Workload Identity annotation
- Isolation audit script
- **HITL gate before kubectl apply**

---

### Example 4: Fix Stuck Helm Release

**You:** "Helm upgrade failed: another operation is in progress"

**Skill provides:**
- Diagnosis: stuck pending-state release
- `helm history` and `helm status` commands
- `kubectl delete secret` fix for pending state
- **HITL gate before secret deletion**
- How to re-run upgrade with `--atomic` flag
- Rollback option

---

### Example 5: Terraform Multi-Env + DB Safety

**You:** "Set up Terraform for dev/staging/prod. Never allow prod DB destruction."

**Skill provides:**
- Directory structure (`environments/dev`, `staging`, `prod`)
- GCS/S3/Azure backend configs (separate per env)
- Provider version pinning (~> semver)
- DB resource with `prevent_destroy = true` + `deletion_protection = true`
- Lifecycle block
- **CRITICAL HITL gate (2-person rule) for prod apply**

---

## Agent Reference

| Agent ID | Specialty | Triggers |
|----------|-----------|----------|
| **001** | Multi-Tenant Isolation | tenant, namespace, RBAC, isolation, NetworkPolicy |
| **002** | Ingress 502 Debug | 502, bad gateway, NGINX, ALB, upstream timeout |
| **003** | Zero-Downtime Deploy | blue-green, canary, rollout, Argo, PDB |
| **004** | CI/CD & GitHub Actions | pipeline, workflow, OPA, bad config |
| **005** | Docker & Containers | dockerfile, multi-stage, ECR, GCR, ACR |
| **006** | Terraform | tfstate, plan, apply, multi-env, DB destroy |
| **007** | Helm | chart, upgrade, rollback, helm lint |
| **008** | Production Grade | P0, incident, debug, playbook, RCA |

---

## Test Cases (Included)

The skill includes 5 comprehensive test cases covering:

1. ✅ 502 debugging on GKE production
2. ✅ Canary deployment on high-traffic EKS
3. ✅ Multi-tenant isolation on GKE
4. ✅ Stuck Helm release recovery
5. ✅ Terraform multi-env with DB protection

All test cases passed during skill development with 100% expectation coverage.

---

## Architecture

```
         User Query
              │
              ▼
      ┌───────────────┐
      │  SKILL.md     │  ← Routes to agent(s)
      └───────┬───────┘
              │
    ┌─────────┴──────────┐
    │   Load References   │
    │  /agents/*.md       │
    │  /shared/skills.md  │
    └─────────┬──────────┘
              │
    ┌─────────▼──────────┐
    │  Determine Env +    │
    │    Risk Level       │
    └─────────┬──────────┘
              │
    ┌─────────▼──────────┐
    │   Generate Output   │
    │  + HITL Gate        │
    │  + Rollback Plan    │
    └────────────────────┘
```

---

## File Structure

```
devops-multi-agent-orchestration/
├── SKILL.md                          ← Main routing & instructions
├── evals/
│   └── evals.json                    ← 5 test cases with expectations
└── references/
    ├── agents/
    │   ├── multi-tenant.md           ← Agent 001
    │   ├── ingress-debug.md          ← Agent 002
    │   ├── zero-downtime.md          ← Agent 003
    │   ├── cicd.md                   ← Agent 004
    │   ├── docker.md                 ← Agent 005
    │   ├── terraform.md              ← Agent 006
    │   ├── helm.md                   ← Agent 007
    │   └── production-grade.md       ← Agent 008
    └── shared/
        └── skills.md                 ← HITL gate, health check, rollback
```

---

## Production Safety

Every response from this skill includes:

✅ **HITL approval gate** (when applicable)  
✅ **Rollback procedure** (< 60 seconds for prod)  
✅ **Pre-flight checklist**  
✅ **Post-action health checks**  
✅ **Cloud-specific variants** (GCP/AWS/Azure)  
✅ **ASCII architecture diagrams**  

---

## Requirements

- **Claude.ai** or **Claude Desktop** with skill support
- Basic familiarity with Kubernetes, cloud platforms, or infrastructure
- No additional dependencies required

---

## Support & Feedback

This skill was built from a comprehensive 11-file, 203KB multi-agent DevOps system. 

For issues or improvements:
- Review the original multi-agent documentation (also included in outputs)
- The skill references contain detailed playbooks and debugging matrices
- All HITL gates are designed to prevent production incidents

---

## Version History

**v1.0** (April 2026)
- Initial release
- 8 specialist agents
- 5 comprehensive test cases
- Full multi-cloud support (GCP, AWS, Azure)
- HITL gates at all production boundaries

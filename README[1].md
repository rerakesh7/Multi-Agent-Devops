# 🤖 DevOps Multi-Agent Orchestration System
### Production-Grade · Multi-Cloud · Human-in-the-Loop

```
╔═══════════════════════════════════════════════════════════════════════════════╗
║            DEVOPS MULTI-AGENT SYSTEM — COMPLETE REFERENCE                     ║
║              GCP · AWS · Azure | GKE · EKS · AKS                             ║
╚═══════════════════════════════════════════════════════════════════════════════╝
```

## System Architecture

```
                         ┌──────────────────────────┐
                         │     HUMAN OPERATOR         │
                         │  (Approval / Override)     │
                         └────────────┬───────────────┘
                                      │ Approvals
                         ┌────────────▼───────────────┐
                         │       ORCHESTRATOR           │
                         │   Intent → Route → Gate      │
                         │   Correlation ID tracking    │
                         │   Audit log every action     │
                         └─────┬───────────────────────┘
                               │
          ┌────────────────────┼──────────────────────────┐
          │                    │                           │
    ┌─────▼──────┐     ┌──────▼──────┐          ┌────────▼──────┐
    │ agent-001  │     │ agent-002   │          │  agent-003    │
    │ Multi-     │     │ 502 Debug   │          │  Zero-Downtime│
    │ Tenant     │     │             │          │  Deploy       │
    └────────────┘     └─────────────┘          └───────────────┘
    ┌────────────┐     ┌─────────────┐          ┌───────────────┐
    │ agent-004  │     │ agent-005   │          │  agent-006    │
    │ CI/CD &    │     │ Docker &    │          │  Terraform    │
    │ GitHub     │     │ Containers  │          │               │
    └────────────┘     └─────────────┘          └───────────────┘
    ┌────────────┐     ┌─────────────┐
    │ agent-007  │     │ agent-008   │
    │ Helm Prod  │     │ Production  │
    │            │     │ Grade       │
    └────────────┘     └─────────────┘
```

## Agent Index

| File | Agent | Responsibility |
|------|-------|----------------|
| `orchestrator/ORCHESTRATOR.md` | Master | Routing, HITL gates, audit |
| `agents/multi-tenant/AGENT.md` | 001 | Namespace isolation (GCP/AWS/Azure) |
| `agents/ingress-debug/AGENT.md` | 002 | 502/gateway error debugging |
| `agents/zero-downtime/AGENT.md` | 003 | Blue/green, canary, rolling deploys |
| `agents/cicd/AGENT.md` | 004 | GitHub Actions, bad-config prevention |
| `agents/docker/AGENT.md` | 005 | Multi-stage Dockerfiles, image lifecycle |
| `agents/terraform/AGENT.md` | 006 | Multi-env IaC, state management, DB safety |
| `agents/helm/AGENT.md` | 007 | Helm upgrades, chart testing, debugging |
| `agents/production-grade/AGENT.md` | 008 | P0 response, playbooks, PIR |
| `skills/SHARED_SKILLS.md` | All | HITL gate, health check, rollback |

## HITL Gate Summary

```
Environment  │ Action Type      │ Approval Required
─────────────┼──────────────────┼────────────────────────────
development  │ Any              │ Auto-approve (logged)
staging      │ Any              │ Slack notification only
production   │ LOW risk         │ 1 engineer approval
production   │ HIGH risk        │ 1 senior engineer approval
production   │ CRITICAL risk    │ 2-person rule (two approvals)
production   │ P0 incident fix  │ IC + manager approval
```

## Emergency Contacts

```
P0 Escalation:  #incidents (Slack) → PagerDuty → on-call
Platform Team:  #platform-team (Slack)
Approvals Log:  #platform-approvals (Slack)
Runbooks:       https://wiki.internal/runbooks/
```

## Quick Reference Commands

```bash
# Trigger specific agent
./orchestrator run agent-002 --env prod --args "ns=api svc=api-service"

# List all pending approvals
./orchestrator approvals list

# Emergency stop all
./orchestrator stop-all --reason "P0 incident"

# Audit log viewer
./orchestrator audit --last 24h --env prod

# Run full health check across all clusters
./orchestrator health-check --all-clusters
```

## File Structure

```
devops-agents/
├── README.md                          ← You are here
├── orchestrator/
│   └── ORCHESTRATOR.md                ← Master routing & HITL
├── agents/
│   ├── multi-tenant/AGENT.md          ← Agent 001
│   ├── ingress-debug/AGENT.md         ← Agent 002
│   ├── zero-downtime/AGENT.md         ← Agent 003
│   ├── cicd/AGENT.md                  ← Agent 004
│   ├── docker/AGENT.md                ← Agent 005
│   ├── terraform/AGENT.md             ← Agent 006
│   ├── helm/AGENT.md                  ← Agent 007
│   └── production-grade/AGENT.md      ← Agent 008
├── skills/
│   └── SHARED_SKILLS.md               ← HITL gate, rollback, health check
├── workflows/                         ← GitHub Actions reusable workflows
├── playbooks/                         ← Incident response playbooks
└── diagrams/                          ← ASCII architecture reference
```

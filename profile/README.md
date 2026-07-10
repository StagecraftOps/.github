# StageCraft

**AI-powered CI/CD intelligence and optimization platform.**

StageCraft watches your GitHub Actions pipelines and does more than tell you something broke. It builds a dependency graph of your workflows, tracks runtime performance and standardization drift, checks governance/compliance posture, finds and root-causes vulnerabilities, and runs a fleet of Bedrock/Claude-backed agents that raise real pull requests — fixing failed runs, remediating dependency vulnerabilities, optimizing slow workflows, and reviewing every PR — instead of just surfacing a dashboard.

## What it does

| Area | Capability |
|---|---|
| **Dependency graph** | Parses workflow YAML (including composite actions, reusable workflow calls, matrix/dispatch fan-out) into a graph across an entire repo or org. |
| **Runtime monitoring** | Deterministic critical-path analysis over real job/step timings — no AI needed to know what's slow. |
| **Standardization** | Diffs workflows against shared templates and clusters recurring patterns to flag drift. |
| **Governance & Compliance** | A Compliance agent checks runs against framework controls (HIPAA/PCI/SOC2); a Governance agent compares behavior against your own uploaded policy docs (pgvector retrieval). |
| **Performance optimization** | Detects bottlenecks and parallelization opportunities, drafts a rewritten workflow, simulates the time saved, and opens a PR on accept. |
| **Vulnerability RCA** *(System agent)* | Trivy/Sonar findings get root-caused, scored against blast radius, and tracked as a GitHub issue. |
| **Vulnerability Remediation** *(Custom agent)* | A publishable Claude Code Action agent that fixes a single finding or a whole repo's findings in dependency order, verified against real npm/PyPI registries. |
| **PR Traces** | A peer-review agent runs on every pull request automatically. |
| **Failure remediation** | Classify → root cause → generate a fix → security-review the fix → PR, all in one agent chain. |
| **Knowledge graph** | Cross-links governance findings, remediation history, and optimization recommendations into one browsable graph. |
| **Pipeline Chat** | Natural-language questions answered over your own pipeline data. |
| **Applications** | Group repos into isolated applications; every page scopes to one application or "all," with no cross-application leakage. |

Agents come in two flavors, shown as two selectable cards in **Agent Fleet**: **System agents** run inside StageCraft itself (Vulnerability RCA, Compliance, Governance, Optimization); **Custom agents** are published as a GitHub Actions workflow into the target repo and dispatched on demand (Vulnerability Remediation, Failure Remediation), so the actual fix work happens as a real, auditable `claude-code-action` run in the repo's own Actions tab.

## Architecture

```
GitHub Actions run / code scanning alert
  → stagecraft-webhook   (verify HMAC signature, publish to SQS)
  → stagecraft-worker    (SQS consumer → Celery: classify → root cause →
                           draft fix → security review → write PR text)
       └─ stagecraft-mcp  (GitHub tools: read workflow YAML/logs, commit,
                           open PRs — structured tool access for the agents)
  → AWS Bedrock (Claude)  (root cause, fix generation, RCA narration, chat)
  → PR / GitHub issue opened on the source repo
  ↕
stagecraft-api      (FastAPI: auth, org/application data, all read/query
                     endpoints, WebSocket live updates)
  ↕
stagecraft-frontend (dashboard — every page above lives here)
```

Custom agents take a second path: the frontend dispatches a `workflow_dispatch` directly against the target repo's own `stagecraft-remediation-agent.yml` (installed there by "Publish to repo"), which runs `anthropics/claude-code-action` in that repo's own Actions — StageCraft just tracks the resulting run.

## Repositories in this workspace

| Directory | Role | Stack | Port |
|---|---|---|---|
| [`stagecraft-api`](stagecraft-api) | Backend API — OAuth, orgs/applications, workflows/runs, remediations, vulnerabilities, optimization, insights, WebSocket relay | FastAPI + async SQLAlchemy, Alembic | 8000 |
| [`stagecraft-frontend`](stagecraft-frontend) | Dashboard — every page in the table above | Angular 18 (standalone, signals), served by nginx.¹ | 3000 |
| [`stagecraft-worker`](stagecraft-worker) | Async analysis — Celery worker + a separate SQS→Celery consumer bridge; the LangGraph agent chains that call Bedrock | Celery + sync SQLAlchemy | — |
| [`stagecraft-webhook`](stagecraft-webhook) | GitHub webhook receiver — signature verification, publishes to SQS. Kept separate so it stays up regardless of API load. | FastAPI | 8001 |
| [`stagecraft-mcp`](stagecraft-mcp) | MCP server exposing GitHub tools (read workflow YAML/logs, commit, open PRs) to the worker's agents | FastMCP | 8010 |
| [`stagecraft-helm`](stagecraft-helm) | Helm charts for all five services above, plus the shared secrets/ingress plumbing | Helm (library chart + umbrella) | — |
| [`stagecraft-infra`](stagecraft-infra) | AWS infrastructure: VPC, EKS, RDS (pgvector), ElastiCache, SQS, IRSA, Secrets Manager | Terraform (2-stage) | — |
| [`pace-stagecraft-monorepo`](pace-stagecraft-monorepo) | **Not a platform service.** A synthetic ~360-service fixture repo used purely as an analysis target to exercise the dependency-graph/standardization/governance features against something realistically large. | — | — |

¹ `stagecraft-frontend`'s repo root still carries an earlier Next.js scaffold; what actually builds and ships is `angular-app/` (see its `Dockerfile`) — treat the Angular app as the current frontend.

`stagecraft-mcp` is called in-cluster over SSE by the worker's agents and isn't needed for most frontend/API work.

## Deployment

```
stagecraft-infra (Terraform, 2 stages)  →  EKS cluster, RDS, ElastiCache, SQS, Secrets Manager
        ↓
stagecraft-helm (helm install)          →  all 5 services on EKS, secrets synced via
                                            External Secrets Operator, one shared ALB
```

Stage 1 of `stagecraft-infra` provisions pure AWS resources; stage 2 needs the live cluster to install the AWS Load Balancer Controller and External Secrets Operator. See each repo's own README for the exact commands and ordering — they're deliberately sequenced and not safely reorderable.

## Status


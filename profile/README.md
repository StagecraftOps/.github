# StageCraft

**Your CI/CD pipeline broke at 2 AM. StageCraft already opened the fix PR.**

Most CI tooling stops at the red ❌ — a dashboard, an alert, maybe a log link. StageCraft keeps going: it builds a dependency graph of your GitHub Actions workflows, finds the critical path, root-causes the failure, drafts a fix, security-reviews its own fix, and opens a real pull request — all before you've finished your coffee.

> **One platform, three jobs:** understand your pipelines (graphs, timings, drift), police them (governance, compliance, vulnerabilities), and *repair* them (a fleet of Bedrock/Claude-backed agents that ship actual PRs).

## ⚡ What it does

| Area | Capability |
|---|---|
| 🕸️ **Dependency graph** | Parses workflow YAML (composite actions, reusable calls, matrix/dispatch fan-out) into a graph across a whole repo or org. |
| ⏱️ **Runtime monitoring** | Deterministic critical-path analysis over real job/step timings — no AI needed to know what's slow. |
| 📐 **Standardization** | Diffs workflows against shared templates and clusters recurring patterns to flag drift. |
| 🏛️ **Governance & Compliance** | A Compliance agent checks runs against HIPAA/PCI/SOC2 controls; a Governance agent checks behavior against *your own* uploaded policy docs (pgvector retrieval). |
| 🚀 **Performance optimization** | Detects bottlenecks, drafts a rewritten workflow, simulates the time saved, and opens a PR on accept. |
| 🔍 **Vulnerability RCA** *(System agent)* | Trivy/Sonar findings get root-caused, scored against blast radius, and tracked as a GitHub issue. |
| 🩹 **Vulnerability Remediation** *(Custom agent)* | A publishable Claude Code Action agent that fixes one finding — or a whole repo's findings in dependency order — verified against real npm/PyPI registries. |
| 👀 **PR Traces** | A peer-review agent runs on every pull request automatically. |
| 🔧 **Failure remediation** | Classify → root cause → generate a fix → security-review the fix → PR, in one agent chain. |
| 🧠 **Knowledge graph** | Cross-links governance findings, remediation history, and optimization recommendations into one browsable graph. |
| 💬 **Pipeline Chat** | Ask questions about your pipelines in plain English, answered over your own data. |
| 🗂️ **Applications** | Group repos into isolated applications — every page scopes cleanly, no cross-application leakage. |

**Two kinds of agents**, two cards in **Agent Fleet**: **System agents** run inside StageCraft (Vulnerability RCA, Compliance, Governance, Optimization). **Custom agents** get published *into your repo* as a GitHub Actions workflow and dispatched on demand (Vulnerability & Failure Remediation) — so the fix work happens as a real, auditable `claude-code-action` run in your own Actions tab, not behind a curtain.

## 🏗️ Architecture

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

Custom agents take a second path: the frontend fires a `workflow_dispatch` straight at the target repo's own `stagecraft-remediation-agent.yml` (installed there via **Publish to repo**), which runs `anthropics/claude-code-action` in that repo's Actions — StageCraft just tracks the resulting run.

## 📦 Repositories

| Repo | Role | Stack | Port |
|---|---|---|---|
| [`stagecraft-api`](https://github.com/StagecraftOps/stagecraft-api) | Backend API — OAuth, orgs/applications, workflows/runs, remediations, vulnerabilities, optimization, insights, WebSocket relay | FastAPI + async SQLAlchemy, Alembic | 8000 |
| [`stagecraft-frontend`](https://github.com/StagecraftOps/stagecraft-frontend) | Dashboard — every page in the table above | Angular 18 (standalone, signals), served by nginx¹ | 3000 |
| [`stagecraft-worker`](https://github.com/StagecraftOps/stagecraft-worker) | Async analysis — Celery worker + SQS→Celery consumer bridge; the LangGraph agent chains that call Bedrock | Celery + sync SQLAlchemy | — |
| [`stagecraft-webhook`](https://github.com/StagecraftOps/stagecraft-webhook) | GitHub webhook receiver — signature verification, publishes to SQS. Separate service so it stays up regardless of API load. | FastAPI | 8001 |
| [`stagecraft-mcp`](https://github.com/StagecraftOps/stagecraft-mcp) | MCP server exposing GitHub tools (read workflow YAML/logs, commit, open PRs) to the worker's agents | FastMCP | 8010 |
| [`stagecraft-helm`](https://github.com/StagecraftOps/stagecraft-helm) | Helm charts for all five services, plus shared secrets/ingress plumbing | Helm (library chart + umbrella) | — |
| [`stagecraft-infra`](https://github.com/StagecraftOps/stagecraft-infra) | AWS infrastructure: VPC, EKS, RDS (pgvector), ElastiCache, SQS, IRSA, Secrets Manager | Terraform (2-stage) | — |
| [`pace-stagecraft-monorepo`](https://github.com/StagecraftOps/pace-stagecraft-monorepo) | **Not a platform service** — a synthetic ~360-service fixture used as a realistically large analysis target for the graph/standardization/governance features. | — | — |

¹ `stagecraft-frontend`'s repo root still carries an earlier Next.js scaffold; what actually builds and ships is `angular-app/` (see its `Dockerfile`).

`stagecraft-mcp` is called in-cluster over SSE by the worker's agents and isn't needed for most frontend/API work.

## 🚢 Deployment

```
stagecraft-infra (Terraform, 2 stages)  →  EKS cluster, RDS, ElastiCache, SQS, Secrets Manager
        ↓
stagecraft-helm (helm install)          →  all 5 services on EKS, secrets synced via
                                            External Secrets Operator, one shared ALB
```

Stage 1 of `stagecraft-infra` provisions pure AWS resources; stage 2 needs the live cluster to install the AWS Load Balancer Controller and External Secrets Operator. Each repo's README has the exact commands — they're deliberately sequenced and not safely reorderable.

---

*Built to make "the pipeline is red" someone else's problem — specifically, the robots'.*

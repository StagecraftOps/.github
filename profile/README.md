<div align="center">

# StageCraft

### AI-Powered CI/CD Intelligence Platform

Transform GitHub Actions, application code, infrastructure, governance policies, and runtime telemetry into an intelligent Knowledge Graph that delivers pipeline visibility, optimization, governance, compliance, and AI-powered DevOps insights.

<p>
  <img src="https://img.shields.io/badge/Python-3.12-blue?logo=python">
  <img src="https://img.shields.io/badge/FastAPI-Backend-009688?logo=fastapi">
  <img src="https://img.shields.io/badge/Neo4j-Knowledge%20Graph-4581C3?logo=neo4j">
  <img src="https://img.shields.io/badge/Next.js-Dashboard-black?logo=next.js">
  <img src="https://img.shields.io/badge/GitHub-Actions-2088FF?logo=githubactions">
  <img src="https://img.shields.io/badge/Kubernetes-Orchestration-326CE5?logo=kubernetes">
</p>

</div>

---

# 🚀 What is StageCraft?

Modern software organizations manage hundreds of services, thousands of workflow executions, infrastructure resources, governance policies, and deployment pipelines. While CI/CD platforms automate delivery, they provide limited visibility into how everything connects.

**StageCraft** transforms engineering data into a centralized **Knowledge Graph**, enabling teams to understand their software delivery ecosystem through intelligent visualization, dependency analysis, governance validation, optimization recommendations, and AI-assisted operational insights.

Instead of viewing workflows as isolated YAML files, StageCraft models the entire engineering ecosystem as a connected graph.

---

# 🏗 Platform Architecture

> The following architecture illustrates the complete StageCraft platform.

<p align="center">

<img src="docs/images/stagecraft-architecture.png" width="100%">

</p>

---

# ⚙️ How StageCraft Works

The platform is organized into seven major layers.

## 1️⃣ Data Sources

StageCraft continuously collects engineering data from multiple sources.

- GitHub Actions
- Application Source Code
- Infrastructure as Code
- Governance Policies
- Runtime Telemetry
- Logs & Metrics

↓

## 2️⃣ Ingestion & Parsing

The ingestion engine:

- Fetches repositories
- Parses workflow YAML
- Extracts jobs and dependencies
- Discovers relationships
- Normalizes metadata
- Validates data

↓

## 3️⃣ Knowledge Graph

All collected information is transformed into a centralized **Neo4j Knowledge Graph**.

### Core Nodes

- Workflow
- Workflow Run
- Job
- Service
- Code Artifact
- Infrastructure Resource
- Governance Policy

### Relationships

- HAS_JOB
- HAS_RUN
- USES
- DEPENDS_ON
- IMPLEMENTS
- GOVERNS
- CALLS
- ACCESSES

This graph becomes the **single source of truth** powering every StageCraft capability.

↓

## 4️⃣ Intelligence Services

Specialized analysis engines continuously evaluate the graph.

### Dependency Intelligence

- Workflow DAG
- Service Dependencies
- Impact Analysis
- Execution Chains

### Performance Intelligence

- Critical Path
- Bottleneck Detection
- Runtime Analytics
- Parallelization Opportunities

### Failure Intelligence

- Root Cause Analysis
- Failure Clustering
- Similar Failure Detection
- Impact Analysis

### Governance

- Policy Validation
- Compliance Checks
- Audit Reports
- Security Recommendations

### Code Intelligence

- API Graph
- Package Dependencies
- Service Coupling
- Technical Health

↓

## 5️⃣ Visualization Layer

StageCraft provides multiple levels of visualization.

### Pipeline Catalog

Search and explore every workflow.

### Template Map

Discover reusable workflow templates.

### Service Chain

Visualize complete service execution flow.

### Job DAG

Inspect workflow execution dependencies.

### Governance Dashboard

Review policy compliance and violations.

### Analytics Dashboard

Monitor DORA metrics, reliability, runtime trends, and optimization insights.

↓

## 6️⃣ Background Analytics

Background workers continuously compute:

- Graph Synchronization
- Runtime Metrics
- Critical Paths
- Pattern Detection
- Optimization Recommendations
- Redis Caching

↓

## 7️⃣ Business Value

StageCraft enables engineering organizations to achieve:

- 🚀 Faster Pipelines
- ✅ Higher Reliability
- 🛡 Stronger Governance
- 👁 Better Visibility
- 💰 Reduced Engineering Cost

---

# 📦 Organization Repositories

The StageCraft platform is built as a collection of modular repositories.

| Repository | Description |
|------------|-------------|
| **pace-stagecraft-monorepo** | Enterprise-scale sample monorepo containing hundreds of services and GitHub Actions workflows |
| **stagecraft-api** | FastAPI backend exposing graph APIs, analytics, and intelligence services |
| **stagecraft-worker** | Celery workers responsible for ingestion, graph synchronization, and background analytics |
| **stagecraft-frontend** | Next.js dashboard providing visualizations and interactive reports |
| **stagecraft-webhook** | GitHub App webhook receiver for real-time repository events |
| **stagecraft-mcp** | Model Context Protocol server providing GitHub tooling for AI agents |
| **stagecraft-infra** | Terraform infrastructure provisioning |
| **stagecraft-helm** | Helm charts for Kubernetes deployment |

---

# ✨ Platform Capabilities

## Dependency Intelligence

- Workflow DAG visualization
- Service dependency mapping
- Impact analysis
- Execution graph exploration

---

## Runtime Intelligence

- Critical path detection
- Build duration analysis
- Workflow performance metrics
- Parallelization recommendations

---

## Governance & Compliance

- Policy validation
- Compliance analysis
- Security recommendations
- Audit reporting

---

## AI-Powered Insights

- Root Cause Analysis (RCA)
- Pipeline optimization
- Intelligent recommendations
- Failure clustering

---

## Analytics

- DORA Metrics
- Reliability reports
- Runtime trends
- Engineering insights

---

# 🛠 Technology Stack

## Backend

- Python
- FastAPI

## Frontend

- Next.js
- React
- React Flow

## Graph Database

- Neo4j

## Background Processing

- Celery
- Redis

## Infrastructure

- Docker
- Kubernetes
- Helm
- Terraform

## CI/CD

- GitHub Actions

---

# 🔄 Platform Workflow

```text
Developer Push / Pull Request
            │
            ▼
     GitHub Actions CI/CD
            │
            ▼
 GitHub Webhook Receiver
            │
            ▼
  StageCraft Ingestion Engine
            │
            ▼
     Neo4j Knowledge Graph
            │
            ▼
    Intelligence Services
            │
    ┌───────┴────────┐
    ▼                ▼
REST APIs      MCP Server
    │
    ▼
Next.js Dashboard
```

---

# 🗺 Roadmap

### Phase 1

- Knowledge Graph Generation
- Dependency Visualization
- Runtime Analytics
- Governance Dashboard

### Phase 2

- Optimization Engine
- AI Root Cause Analysis
- Recommendation Engine
- Pipeline Simulation

### Phase 3

- Multi-CI Platform Support
- Autonomous AI Agents
- Predictive Failure Detection
- Enterprise Multi-Tenancy
- Cost Optimization
- Real-Time Event Streaming

---



<div align="center">

### Build Intelligent CI/CD Systems with StageCraft

**Knowledge Graphs • DevOps Intelligence • AI-Powered Engineering**

</div>

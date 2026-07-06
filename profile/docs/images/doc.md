# Stagecraft — How It Works (Demo Reference)

Everything below is verified against the actual live code, not a design doc — file paths and endpoints are exact.

---

## 1. Architecture in one pass

Five services, two databases, one queue as the nervous system:

| Service | Role |
|---|---|
| `stagecraft-frontend` | Next.js dashboard — every tab below is a page in here |
| `stagecraft-api` | FastAPI — auth, all REST routes, enqueues work onto SQS |
| `stagecraft-worker` | Celery — every AI agent and graph builder lives here |
| `stagecraft-mcp` | Tool server the AI agents call for GitHub reads and graph queries |
| `stagecraft-webhook` | Receives GitHub events, drops them on SQS |

**Postgres** holds everything relational — runs, remediations, findings, templates, users, chat history. **Neo4j** holds the dependency/knowledge graph as real nodes and relationships (see §4). **Bedrock** is the model layer: **Anthropic's Sonnet model** (`bedrock_model_id = us.anthropic.claude-sonnet-4-6`) does all the reasoning/generation calls; **Amazon Titan** does text embeddings for RAG search.

**Data flow for a failed run** (the core, oldest path):
```
GitHub webhook → stagecraft-webhook → SQS → stagecraft-worker's remediation
LangGraph (classify → root cause → fix → security check → PR-or-human-gate →
confidence score) → written to `remediations` table → surfaced on the
Remediation tab
```

---

## 2. Tab-by-tab

### Overview section

#### Dashboard (`/dashboard`)
Landing page. Shows 4 stat cards (Total Workflows, Active Runs, Failed Today, AI Fixes Raised — all computed client-side from already-fetched data, not separate endpoints), a "Recent Workflow Runs" table (last 10, `GET /api/v1/runs/recent`), and a "Recent Remediations" feed (last 3, `GET /api/v1/remediations/`). Subscribes to a WebSocket (`useWebSocket`) — a `run_update` or `remediation_created`/`remediation_updated` event live-invalidates the relevant query, so new failures/fixes appear without a manual refresh. Clicking "View all" just navigates to `/runs` or `/remediation`.

#### Analytics (`/analytics`)
Pure SQL aggregates, no AI, no hardcoded numbers. Backed by `useAnalytics` hook → one endpoint that returns total runs, a 30-day run trend (success/failed per day, feeds the line chart), top-10 failing repos (bar chart), a 3-way completed-run outcome split (success/failure/other — "other" = cancelled/skipped/timed-out/startup-failure), and remediation timing (PRs raised, avg time-to-fix-suggestion, avg time-to-PR). Nothing here is clickable beyond navigation; it's read-only reporting.

#### Performance (`/performance`)
"Pure duration ranking, no AI involved" (literally the page's own subtitle). Two panels: **Longest-Running Jobs** (bar chart, `fetchLongestJobs`) and **Longest-Running Workflows** (list, `fetchLongestWorkflows`) — both ranked by raw duration from synced run-timing data. Clicking a workflow in the list navigates to `/runs/{workflow_run_id}` (that run's detail page, logs included).

### Pipelines section

#### Runs (`/runs`)
Every workflow run across every connected repo — "replacing the per-repo Actions tab" per its own description. Filters: repo, status (queued/in_progress/completed), conclusion (success/failure/cancelled). Paginated 25/page via `GET /api/v1/runs` with `org_login`/`repo_name`/`status`/`conclusion`/`limit`/`offset` params. Shows a live "Syncing historical runs…" banner while an org's initial backfill (`backfill_org_runs_task`, triggered once on GitHub App install) is still in progress. Clicking a row goes to `/runs/{run_id}` — the single-run detail page with full logs.

#### Workflows (`/workflows`)
Every GitHub Actions workflow file across the org — `GET` via `fetchWorkflowsByOrg`, filterable by name/repo search and by state (active/failed/disabled — GitHub's own workflow state, not a Stagecraft concept). Clicking a card goes to `/workflows/{owner}/{repo}` for that specific workflow's run history.

#### Dependency Graph (`/dependency-graph`)
**FR-1.** The worker fetches every `.github/workflows/*.yml` (plus `orchestrator.yaml`/`service-config.json` if present) straight from GitHub and parses the YAML — **no AI, pure structural parsing** (`app/analysis/workflow_parser.py`, `graph_builder.py`). Produces nodes (`workflow`, `job`, `reusable_workflow`, `composite_action`, `service`, `external_repo`) and edges (`needs`, `uses_reusable`, `workflow_run_trigger`, `orchestrator_service_dep`, `repository_dispatch`, etc.), dual-written to Postgres (`graph_nodes`/`graph_edges`) and Neo4j.

Pick a repo → **Rebuild graph** (`POST /orgs/{org}/repos/{repo}/dependency-graph/build`) → enqueues `build_dependency_graph_task`. The page (`GET /orgs/{org}/repos/{repo}/dependency-graph`) shows a **two-level view**: default is a collapsed workflow-to-workflow directory (circular layout — this is a hub-and-spoke graph, dozens of workflows calling a handful of shared templates, so a ring reads cleaner than a hierarchy). Click a workflow to drill into its own jobs + `needs` chain (dagre hierarchical layout — a real DAG); dashed boxes are external references to other workflows, click a resolvable one to jump straight there.

### AI Agents section

#### Remediation (`/remediation`, detail at `/remediation/{id}`)
The oldest, most-exercised path. Every row in `remediations` is one failed-run analysis. The list page (`GET /api/v1/remediations/`) shows repo/workflow/root-cause preview/status/PR link/date. Clicking a row → `/remediation/{id}` (`GET /api/v1/remediations/{id}`) shows:
- **Root cause** (plain text from the LLM)
- **Suggested Fix** — full YAML diff, copyable
- **Timeline** — Failure detected → Bedrock analysis → Suggested fix ready → PR raised
- **Raise PR** button (only enabled once `status === 'analyzed'` and a YAML fix exists) → `POST /api/v1/remediations/{id}/raise-pr` — opens a **real GitHub PR** with the suggested fix
- Auto-polls every 3s while `status` is `pending`/`analyzing`

**The actual LangGraph pipeline** (`app/agents/graph.py`), one node per stage:
```
classify_failure → analyse_root_cause → generate_fix → review_security
   → [confidence gate] → blocked (human review) OR pr_writer → score_confidence → END
```
`classify_failure` buckets into one of 8 fixed categories (`DEPENDENCY_VERSION`, `AUTH_FAILURE`, `NETWORK_TIMEOUT`, `CONFIG_ERROR`, `TEST_FAILURE`, `BUILD_ERROR`, `LINT_ERROR`, `PERMISSION_ERROR`) or `UNKNOWN` if it can't confidently match one. Every stage appends to `agent_trace` (visible via the API response, not currently shown in the UI). `score_confidence` (0-100) gates whether the fix auto-proceeds or stops for human review.

> **Known stale text**: the "Analyzing" placeholder on the detail page still says *"Amazon Nova is reading the failure logs"* — the model was migrated to Anthropic's Sonnet a while back (`bedrock_model_id` confirms this) and this string was never updated. Cosmetic, but worth fixing before a demo since someone might ask "wait, I thought this used Claude/Sonnet?"

#### Peer Review (`/pr-reviews`)
Runs on **every** PR opened/updated/reopened in a connected org — not gated on touching CI files (a common misconception; `detect_workflow_changes` only adds CI-file context to the LLM prompt, it never skips the review). Webhook → SQS (`pull_request` event) → `process_pull_request` → `peer_review` LangGraph (`detect_changes → review → END`) → writes to `pr_reviews`, surfaced here with a risk score (0-10), summary, and findings. Click a row to expand the full summary/findings/agent-trace (previously truncated at 80 chars with no way to see the rest — fixed this session).

#### Pipeline Chat (`/chat`)
`POST /api/v1/chat/`. A cheap classification call first buckets the question as **COUNT** (a literal counting/ranking question, e.g. "which repo fails most") or **INVESTIGATE** (anything needing reasoning — root causes, comparisons, trends). COUNT answers straight from a SQL `GROUP BY` against `workflow_runs` — fast, exact, shown with an expandable "Show SQL" block. INVESTIGATE hands off via HTTP to the worker's **Investigator Agent** (`app/agents/investigator.py`) — a bounded (max 5 rounds) Bedrock tool-calling loop with three tools:
- `search_remediations` — pgvector semantic/filtered search over remediation history
- `get_run_logs` — pulls one specific run's raw logs if the summary isn't enough
- `query_graph` — **real graph traversal**, not text search (see §5) — "what does this workflow depend on," "what governs it," "what's its failure history"

Chat history persists in `chat_messages` (last 20 turns loaded per request) so follow-up questions have context.

### Quality section

#### Governance (`/governance`)
Two independent modes, one shared judging step:
- **Document mode**: upload a policy doc (`.txt`/`.docx`/`.pdf`, `POST /orgs/{org}/repos/{repo}/governance/documents`) → an LLM extracts structured requirements into `governance_documents.structured_requirements` (JSONB) and embeds the raw text into `log_embeddings` (pgvector, 1024-dim Titan embeddings) → **Analyze** on that doc (`POST .../governance/analyze` with `mode: "document"`) retrieves the relevant policy chunks *and* real graph structure (GraphRAG, see §5) before judging each workflow compliant/gap/not-applicable.
- **Framework mode**: pick HIPAA/PCI/SOC2 from a dropdown, no document needed — judged against a fixed control list, same GraphRAG-augmented judging step.

Both write to `compliance_findings` (`requirement_id`, `status`, `finding_detail`, `remediation_suggestion`, `severity`), shown in the Findings panel. A full run against a large monorepo (~111 workflow files) takes 20-35 minutes — one sequential Bedrock call per file. That's expected, not stuck.

#### Standardization (`/standardization`)
**Where the standard template is actually stored**: `workflow_templates` table (`id`, `org_login`, `name`, `description`, `template_yaml` — the raw YAML text — `is_active`, `created_by`, timestamps). **Register template** button → `POST /orgs/{org}/templates` with `{name, description, template_yaml}` — that's it, no GitHub write, just a Postgres row.

**Analyze** (`POST .../standardization/analyze`) runs two independent things:
1. **Template diff (FR-3)**: every workflow in the repo gets diffed against every active template — deterministic set comparison of `uses:` components (missing/extra/version-drift) into a 0-100 adoption score, written to `template_diffs`. **LLM layer** (added this cycle): for any non-fully-compliant diff, `narrate_template_diff` explains *why* the gap matters (real risk vs. harmless project quirk) — shown as an italic note under the diff.
2. **Pattern discovery (FR-4)**: `find_repeated_patterns` clusters jobs by exact component-signature hash (≥3 occurrences), written to `pattern_clusters`. **LLM layer** (added this cycle): `find_near_miss_groups` groups jobs the exact hash missed (similar but not byte-identical), and `judge_pattern_cluster` asks Bedrock whether they're genuinely the same pattern — if so, it also **drafts the actual reusable-workflow YAML**, shown as an "AI-detected" badge with an expandable draft.

Both LLM calls are best-effort — a Bedrock hiccup just leaves the deterministic result as-is.

#### Optimization (`/optimization`)
`POST /orgs/{org}/repos/{repo}/optimization/analyze` with `{workflow_file, ref}`. Three things: **bottleneck detection** (a job's duration vs. its own historical p90, and whether it sits on the critical path — reads job-edge structure straight from Neo4j when `GRAPH_BACKEND=neo4j`), **parallelization advisor** (flags `needs` edges with no real data dependency — artificially serialized jobs), and a **simulation agent** that proposes a draft YAML change and estimates the resulting critical-path duration, written to `optimization_recommendations` + `simulation_runs`. Needs synced run-timing history to have anything to compare against — "no runs" on a fresh workflow is expected, not broken.

#### Knowledge Graph (`/knowledge-graph`)
**FR-11.** Cross-links three Postgres tables — `compliance_findings`, `remediations`, `optimization_recommendations` — onto the dependency graph's real Workflow nodes, as `GovernanceRule`/`Failure`/`RuntimeMetric` nodes connected via `GOVERNS`/`CAUSED_BY`/`MEASURED_BY` edges. This is the graph GraphRAG reads from (§5). Built by `POST /orgs/{org}/knowledge-graph/build` (also auto-triggered now whenever a new classified remediation, finding, or recommendation lands — see below).

**Every edge here points FROM a rule/failure/metric INTO a workflow** — there is no workflow-to-workflow edge in this data at all (unlike the dependency graph, where workflows really do call each other). Default view is a **directory**: one node per workflow, no edges (nothing structural to draw — see the long back-and-forth on this exact question), with a count badge (e.g. "14 rules, 1 failure") and a **red outline on any workflow with a failure attached**. Click a workflow to drill into just its own rules/failures/metrics. Click a red **Failure** node to jump straight to `/remediation/{id}` for the full analysis.

#### Settings (`/settings`)
Lists connected GitHub orgs (`GET /orgs`) with a remove button (`DELETE`, just un-tracks it — doesn't uninstall the GitHub App) and an **Install GitHub App** button that redirects to GitHub's install flow with the session cookie carried through. New orgs appear automatically once the App is installed (webhook-driven), no manual "add org" step.

---

## 3. Where things are actually stored (quick reference)

| Thing | Table | Key columns |
|---|---|---|
| Standard/approved templates | `workflow_templates` | `template_yaml` (raw text), `is_active` |
| Template diff results | `template_diffs` | `diff_summary` (JSONB — includes `narrative` when non-compliant) |
| Reusable-pattern clusters | `pattern_clusters` | `pattern_signature` (JSONB — includes `match_type`, `draft_template_yaml` for AI-detected ones) |
| Governance policy docs | `governance_documents` | `raw_text`, `structured_requirements` (JSONB) |
| Compliance/governance findings | `compliance_findings` | `status`, `finding_detail`, `severity` |
| Remediation analyses | `remediations` | `failure_category`, `root_cause`, `suggested_yaml`, `confidence_score` |
| PR peer reviews | `pr_reviews` | `risk_score`, `findings`, `review_summary` |
| Optimization recommendations | `optimization_recommendations` + `simulation_runs` | `recommendation_type`, `proposed_yaml_diff` |
| RAG embeddings (policy text + remediation context) | `log_embeddings` | `embedding` (pgvector, 1024-dim Titan) |
| Chat history | `chat_messages` | `role`, `content` |
| Dependency/knowledge graph (Postgres side, legacy read path) | `graphs`/`graph_nodes`/`graph_edges` | — |

---

## 4. How Neo4j is actually used

**Dual-write, single cutover flag.** Every dependency-graph and knowledge-graph write goes to Postgres *and* (when `GRAPH_DUAL_WRITE_NEO4J` is on) to Neo4j, as real `:GraphNode` nodes with type-specific labels (`:Workflow`, `:Job`, `:GovernanceRule`, `:Failure`, etc.) and typed relationships (`NEEDS`, `USES_REUSABLE`, `GOVERNS`, `CAUSED_BY`, ...). Postgres remains the permanent safety net; Neo4j is additive. A separate flag, `GRAPH_BACKEND` (`postgres`|`neo4j`), controls which one the **read path** actually serves from — currently `neo4j` in production for both the dependency-graph and knowledge-graph API routes.

**Identity scheme**: every node's Neo4j identity is `{org_login}::{repo_scope}::{external_key}` — `repo_scope` is the real repo name for repo-owned types (`workflow`, `job`, `composite_action`) so two repos never collide, but is empty for org-wide types (`service`, `external_repo`, and — since a fix this session — `reusable_workflow`, so two repos calling the identical external shared template correctly merge onto one node instead of getting duplicates).

**Why Neo4j at all, versus just Postgres**: the dependency/knowledge graph is fundamentally a traversal problem ("what depends on this," "what governs this workflow and what's failed on it") — Postgres can do that with recursive CTEs, but a real graph database does it as a first-class, indexed operation, which is what makes `query_graph` (used by both Optimization's job-edge lookups and the chat Investigator) fast and simple instead of a hand-rolled recursive query.

---

## 5. How GraphRAG is actually wired

**GraphRAG = vector search for policy *text* + Neo4j traversal for graph *structure*, combined in one prompt.** Two separate pieces:

1. **`retrieve_graph_context`** (`app/agents/graph_context.py`) — shared by the Governance and Compliance LangGraph agents. Before judging a workflow, it runs one Cypher query against Neo4j:
   ```cypher
   MATCH (w:GraphNode:Workflow {org_login, repo_name, workflow_file})
   OPTIONAL MATCH (rule:GovernanceRule)-[:GOVERNS]->(w)
   OPTIONAL MATCH (fail:Failure)-[:CAUSED_BY]->(w)
   OPTIONAL MATCH (w)-[:NEEDS|USES_REUSABLE|USES_COMPOSITE]->(dep)
   ```
   pulling every rule already governing this workflow, its known failure history, and its dependencies — folded into the judging prompt as "Already governed by: ...", "Known failure history: ...", "Depends on: ...". Gated on `GRAPH_DUAL_WRITE_NEO4J` (not `GRAPH_BACKEND` — it only needs Neo4j to *have* the data, independent of whether the graph API routes have cut over to reading from it).

2. **`query_graph` MCP tool** (`stagecraft-mcp/src/server.py`) — the same idea, exposed as a tool the chat Investigator Agent can call mid-conversation. Four fixed traversal shapes: `depends_on` (what this workflow calls), `depended_on_by` (what calls it), `governance` (rules linked to it), `failures` (failure history). Proxies `worker → mcp → api`'s `/internal/graph/query` → Neo4j, so "what does ci-auth-service depend on" gets a structurally correct answer instead of an LLM guessing from text.

Both pieces read the **same underlying Neo4j data** the Knowledge Graph tab visualizes — the tab is the human-readable view of exactly what GraphRAG is feeding into the agents.

---

## 6. Quick gaps to know about before a demo

- Remediation detail page's loading text says "Amazon Nova" — stale, real model is Anthropic's Sonnet (not blocking, just inconsistent if someone reads closely).
- Bedrock bearer API key has expired mid-session multiple times historically — if a remediation/governance run stalls, this is the first thing to check.
- MCP is pinned to 1 replica (SSE session affinity) — a crash means a short chat/tool outage.
- No autoscaling anywhere in the cluster; no staging environment (all testing is against the one live cluster).

# Runbook Generator — Workflow Plan

## Overview
Operational runbook pipeline for SRE teams. Pulls incident history from GitHub issues via `gh` CLI, analyzes common failure patterns, generates step-by-step runbooks with diagnostic commands and remediation procedures, reviews for completeness, and publishes to the repository wiki. Tracks runbook coverage across services and identifies gaps.

## Why This Example Matters
Every SRE team needs runbooks but they're tedious to write and always out of date. This shows AO automating operational documentation from real incident data — using only gh CLI and filesystem tools that exist today. Demonstrates AO for DevOps/SRE use cases beyond coding.

---

## Agents

| Agent | Model | Role |
|---|---|---|
| **incident-analyst** | claude-haiku-4-5 | Fast pass: fetch and parse GitHub issues labeled as incidents, extract failure patterns, group by service/severity |
| **procedure-writer** | claude-sonnet-4-6 | Write detailed runbook procedures with diagnostic commands, remediation steps, escalation paths, and rollback instructions |
| **runbook-reviewer** | claude-sonnet-4-6 | Review runbooks for completeness, accuracy, command correctness, and SRE best practices |
| **wiki-publisher** | claude-sonnet-4-6 | Publish finalized runbooks to GitHub wiki, update index page, generate coverage report |

## Phase Pipeline

```
1. fetch-incidents          (command phase — gh CLI)
   ↓
2. analyze-patterns         (agent phase — incident-analyst)
   ↓
3. prioritize-runbooks      (agent phase — decision: which runbooks to generate)
   ↓
4. write-runbooks           (agent phase — procedure-writer)
   ↓
5. review-runbooks          (agent phase — decision: approve/rework/reject)
   ↓  rework → back to write-runbooks
6. publish-to-wiki          (agent phase — wiki-publisher)
   ↓
7. coverage-report          (agent phase — wiki-publisher)
```

### Phase Details

#### 1. fetch-incidents (command phase)
- **What:** Pull incident-labeled issues from the target GitHub repo
- **How:** Command phase using `gh` CLI
- **Commands:**
  - `gh issue list --label incident,outage,postmortem --state all --limit 200 --json number,title,body,labels,createdAt,closedAt,comments`
  - Save raw output to `data/raw-incidents.json`
- **Output:** Raw incident JSON data

#### 2. analyze-patterns (agent phase)
- **Agent:** incident-analyst (haiku — fast structured extraction)
- **What:** Parse incident issues, extract:
  - Affected services/components
  - Root cause categories (deployment, infrastructure, dependency, config, capacity)
  - Severity levels (P1-P4) from labels or content
  - Common failure patterns and recurring issues
  - Time-to-resolution stats
- **Input:** `data/raw-incidents.json`
- **Output:** `data/incident-analysis.json` with structure:
  ```json
  {
    "services": ["api-gateway", "auth-service", "database", "cdn"],
    "patterns": [
      {
        "id": "pattern-001",
        "name": "Database connection pool exhaustion",
        "service": "database",
        "severity": "P1",
        "frequency": 5,
        "incidents": ["#42", "#87", "#103"],
        "root_cause": "Connection leak in ORM under high concurrency",
        "symptoms": ["5xx errors spike", "latency > 2s", "connection timeout logs"],
        "resolution_summary": "Restart pods, increase pool size, deploy fix"
      }
    ],
    "coverage_gaps": ["auth-service has 3 P1 incidents but no runbook"],
    "stats": { "total_incidents": 47, "p1": 5, "p2": 12, "p3": 20, "p4": 10 }
  }
  ```
- **MCP:** filesystem (read/write), sequential-thinking (pattern reasoning)

#### 3. prioritize-runbooks (agent phase — decision)
- **Agent:** incident-analyst
- **What:** Decide which runbooks to generate based on incident frequency, severity, and existing coverage
- **Decision contract:** `{ verdict: "proceed" | "insufficient-data", reasoning: "...", runbooks_to_generate: [...] }`
- **Priority rules:**
  - P1 patterns with no existing runbook → highest priority
  - Recurring patterns (3+ incidents) → high priority
  - Services with zero runbook coverage → medium priority
  - Skip patterns with < 2 incidents unless P1
- **Output:** `data/runbook-queue.json` — ordered list of runbooks to generate
- **MCP:** filesystem, sequential-thinking

#### 4. write-runbooks (agent phase)
- **Agent:** procedure-writer (sonnet — needs careful technical writing)
- **What:** For each queued pattern, generate a complete operational runbook:
  ```markdown
  # Runbook: [Pattern Name]
  
  **Service:** api-gateway
  **Severity:** P1
  **Last Updated:** 2026-03-31
  **Owner:** SRE Team
  
  ## Symptoms
  - Alert: `api_error_rate > 5%` fires
  - Users report 502/503 errors
  - Dashboard: grafana.internal/d/api-health shows red
  
  ## Diagnostic Steps
  1. Check pod status:
     ```bash
     kubectl get pods -n api-gateway -l app=api-gateway
     ```
  2. Check recent deployments:
     ```bash
     kubectl rollout history deployment/api-gateway -n api-gateway
     ```
  3. Check error logs:
     ```bash
     kubectl logs -n api-gateway -l app=api-gateway --tail=100 | grep ERROR
     ```
  
  ## Remediation
  ### Quick Fix (restore service)
  1. Rollback last deployment:
     ```bash
     kubectl rollout undo deployment/api-gateway -n api-gateway
     ```
  
  ### Root Cause Fix
  1. [Specific steps based on incident pattern]
  
  ## Escalation
  - If not resolved in 15 minutes → page on-call lead
  - If data loss suspected → engage database team
  
  ## Rollback Plan
  [Steps to undo any changes made during remediation]
  
  ## Related Incidents
  - #42 (2026-01-15) — first occurrence
  - #87 (2026-02-20) — recurrence after config change
  ```
- **Input:** `data/runbook-queue.json`, `data/incident-analysis.json`
- **Output:** `output/runbooks/<service>-<pattern-slug>.md` for each runbook
- **MCP:** filesystem (read analysis, write runbooks), sequential-thinking (reason about procedures), github (look up incident details)

#### 5. review-runbooks (agent phase — decision)
- **Agent:** runbook-reviewer (sonnet — quality gate)
- **What:** Review each runbook for:
  - **Completeness:** All sections present (symptoms, diagnostics, remediation, escalation, rollback)
  - **Command accuracy:** All CLI commands are syntactically valid
  - **Actionability:** Steps are specific enough for an on-call engineer at 3 AM
  - **Consistency:** Formatting, severity labels, escalation paths consistent across runbooks
- **Decision contract:** `{ verdict: "approve" | "rework" | "reject", reasoning: "...", issues: [...] }`
- **Rework:** Returns to write-runbooks with specific feedback
- **Output:** `data/review-results.json`
- **MCP:** filesystem (read runbooks), sequential-thinking (quality reasoning)

#### 6. publish-to-wiki (agent phase)
- **Agent:** wiki-publisher (sonnet)
- **What:** Publish approved runbooks to the GitHub repository wiki:
  - Create/update wiki pages via `gh api` or git operations on the wiki repo
  - Update the wiki Home/index page with a table of all runbooks
  - Add cross-links between related runbooks
  - Commit and push wiki changes
- **MCP:** github (wiki operations), filesystem (read runbooks)

#### 7. coverage-report (agent phase)
- **Agent:** wiki-publisher
- **What:** Generate a coverage report showing:
  - Services with runbook coverage vs gaps
  - Severity distribution of covered vs uncovered patterns
  - Staleness report (runbooks not updated in 30+ days)
  - Recommendations for next runbook batch
- **Output:** `output/coverage-report.md` and `output/coverage-summary.json`
- **MCP:** filesystem (write report), github (check existing wiki pages)

---

## MCP Servers

| Server | Package | Purpose |
|---|---|---|
| **filesystem** | `@modelcontextprotocol/server-filesystem` | Read/write incident data, runbook files, reports |
| **github** | `gh-cli-mcp` | Fetch issues, interact with wiki, create releases |
| **sequential-thinking** | `@modelcontextprotocol/server-sequential-thinking` | Complex reasoning for pattern analysis and procedure design |

## Workflow Configuration

- **Rework loop:** review-runbooks can send back to write-runbooks up to 3 times
- **Schedule:** Weekly cron (`0 9 * * 1`) to refresh runbooks from new incidents
- **Post-success:** Commit output files, no PR needed (wiki is the primary output)

## Directory Structure

```
examples/runbook-generator/
├── .ao/workflows/
│   ├── agents.yaml
│   ├── phases.yaml
│   ├── workflows.yaml
│   ├── mcp-servers.yaml
│   └── schedules.yaml
├── config/
│   └── settings.yaml          # Target repo, labels, severity mapping
├── data/                      # Working data (gitignored)
│   ├── raw-incidents.json
│   ├── incident-analysis.json
│   ├── runbook-queue.json
│   └── review-results.json
├── output/
│   └── runbooks/              # Generated runbook markdown files
│       └── coverage-report.md
├── templates/
│   └── runbook-template.md    # Template for consistent formatting
├── sample-data/
│   └── sample-incidents.json  # Demo data for testing without a real repo
├── CLAUDE.md
├── PLAN.md
└── README.md
```

## Sample Config (config/settings.yaml)

```yaml
target_repo: "myorg/myapp"
incident_labels:
  - incident
  - outage
  - postmortem
  - sev1
  - sev2
severity_mapping:
  sev1: P1
  sev2: P2
  incident: P2
  outage: P1
  postmortem: P2
services:
  - api-gateway
  - auth-service
  - database
  - worker
  - cdn
wiki_prefix: "Runbook"
coverage_threshold: 80
```

# Runbook Generator

Automated operational runbook pipeline for SRE teams. Pulls incident history from GitHub issues, analyzes failure patterns, generates step-by-step runbooks with diagnostic commands and remediation procedures, reviews for quality, and publishes to the repository wiki.

## Workflow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    RUNBOOK GENERATOR PIPELINE                       │
└─────────────────────────────────────────────────────────────────────┘

  ┌──────────────────┐
  │  fetch-incidents │  gh issue list --label incident,outage,sev1,sev2
  │  (command phase) │  → data/raw-incidents.json
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────┐
  │ analyze-patterns │  incident-analyst (haiku) — extract services,
  │  (agent phase)   │  patterns, severity, root causes, TTR stats
  └────────┬─────────┘  → data/incident-analysis.json
           │
           ▼
  ┌────────────────────┐
  │prioritize-runbooks │  incident-analyst — rank by severity + frequency
  │  (decision phase)  │  → data/runbook-queue.json
  └────────┬───────────┘
           │ insufficient-data ──► fetch-incidents
           │ proceed
           ▼
  ┌──────────────────┐
  │  write-runbooks  │  procedure-writer (sonnet) — detailed runbooks
  │  (agent phase)   │  with exact CLI commands, escalation paths
  └────────┬─────────┘  → output/runbooks/*.md
           │
           ▼
  ┌──────────────────┐
  │ review-runbooks  │  runbook-reviewer (sonnet) — completeness,
  │  (decision phase)│  command accuracy, actionability check
  └────────┬─────────┘  → data/review-results.json
           │ rework (up to 3x) ──► write-runbooks
           │ reject ─────────────► analyze-patterns
           │ approve
           ▼
  ┌──────────────────┐
  │ publish-to-wiki  │  wiki-publisher (sonnet) — clone wiki repo,
  │  (agent phase)   │  copy runbooks, update Home.md index, push
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────┐
  │ coverage-report  │  wiki-publisher — coverage %, gap analysis,
  │  (agent phase)   │  staleness warnings, next-batch recommendations
  └──────────────────┘  → output/coverage-report.md
                          output/coverage-summary.json

  ─────────────────────────────────────────────────────
  SCHEDULE: Every Monday 09:00 → runbook-refresh workflow
```

## Quick Start

```bash
# 1. Configure your target GitHub repo
vim config/settings.yaml   # set target_repo: "myorg/myapp"

# 2. Set required environment variables
export GITHUB_TOKEN=ghp_...

# 3. Start the daemon and trigger the workflow
cd examples/runbook-generator
ao daemon start
ao queue enqueue \
  --title "runbook-generator" \
  --description "Generate runbooks from incident history" \
  --workflow-ref runbook-generator

# 4. Watch it run
ao daemon stream --pretty

# 5. Find your runbooks
ls output/runbooks/
cat output/coverage-report.md
```

**No real GitHub repo?** Use the included sample data:
```bash
# Copy sample incidents to data/ to skip the fetch step
cp sample-data/sample-incidents.json data/raw-incidents.json
# Then trigger starting from analyze-patterns phase
```

## Agents

| Agent | Model | Role |
|---|---|---|
| **incident-analyst** | `claude-haiku-4-5` | Fast structured extraction — parse incident issues, identify failure patterns, group by service and severity, build the runbook priority queue |
| **procedure-writer** | `claude-sonnet-4-6` | Write production-quality runbooks with exact diagnostic commands, dual-track remediation (quick fix + root cause), and escalation paths |
| **runbook-reviewer** | `claude-sonnet-4-6` | Quality gate — check completeness, command syntax, actionability, and consistency before publishing |
| **wiki-publisher** | `claude-sonnet-4-6` | Publish to GitHub wiki, maintain the index page, generate coverage reports with gap analysis |

## AO Features Demonstrated

| Feature | Where |
|---|---|
| **Command phases** | `fetch-incidents` — gh CLI to pull issues as JSON |
| **Multi-agent pipeline** | 4 specialized agents in sequence |
| **Decision routing** | `prioritize-runbooks` routes to `fetch-incidents` on insufficient data |
| **Rework loops** | `review-runbooks` sends back to `write-runbooks` up to 3× |
| **Decision contracts** | Both `prioritize-runbooks` and `review-runbooks` use structured verdicts |
| **Output contracts** | Structured JSON files passed between phases |
| **Scheduled workflows** | Weekly Monday 09:00 cron refresh via `runbook-refresh` workflow |
| **Mixed models** | haiku for fast extraction, sonnet for careful writing and review |
| **Multiple MCP servers** | filesystem + sequential-thinking + gh-cli-mcp |
| **post_success commit** | Auto-commits output files after successful run |

## Requirements

### Environment Variables

| Variable | Required | Purpose |
|---|---|---|
| `GITHUB_TOKEN` | Yes | GitHub API access for issue fetching and wiki publishing |

### Tools

- `gh` CLI installed and authenticated (`gh auth login`)
- Node.js 18+ (for npx MCP server commands)
- Python 3.x (for JSON merging in fetch-incidents command phase)
- `ao` daemon installed and configured

### GitHub Repository Setup

Your target repo (set in `config/settings.yaml`) should have:
- Issues labeled with some of: `incident`, `outage`, `postmortem`, `sev1`, `sev2`
- Wiki enabled (Settings → Features → Wikis)

## Directory Structure

```
examples/runbook-generator/
├── .ao/workflows/
│   ├── agents.yaml          # 4 agents: analyst, writer, reviewer, publisher
│   ├── phases.yaml          # 7 phases including decision phases and command phase
│   ├── workflows.yaml       # runbook-generator + runbook-refresh workflows
│   ├── mcp-servers.yaml     # filesystem, sequential-thinking, github MCPs
│   └── schedules.yaml       # Weekly Monday refresh schedule
├── config/
│   └── settings.yaml        # Target repo, labels, severity mapping
├── data/                    # Working data (auto-generated, gitignored)
│   ├── raw-incidents.json
│   ├── incident-analysis.json
│   ├── runbook-queue.json
│   └── review-results.json
├── output/
│   └── runbooks/            # Generated runbook markdown files
│       └── coverage-report.md
├── sample-data/
│   └── sample-incidents.json  # Demo data for testing without a real GitHub repo
├── templates/
│   └── runbook-template.md    # Formatting baseline for procedure-writer
├── CLAUDE.md
└── README.md
```

## Example Output

Running against a real repo with 50 incident issues produces:

```
$ ls output/runbooks/
api-gateway-connection-pool-exhaustion.md
api-gateway-cascading-timeout-failure.md
auth-service-jwt-key-rotation-lockout.md
auth-service-rate-limiter-misconfiguration.md
database-connection-pool-exhaustion.md
database-slow-query-cascade.md
worker-redis-oom-queue-backlog.md
coverage-report.md

$ cat output/coverage-summary.json
{
  "overall_coverage_pct": 87.5,
  "total_patterns": 8,
  "covered_patterns": 7,
  "p1_coverage_pct": 100.0,
  ...
}
```

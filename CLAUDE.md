# Runbook Generator — Agent Context

## What This Repo Does

This is an AO workflow that automates SRE runbook generation. It:
1. Fetches incident-labeled GitHub issues from a target repository
2. Analyzes the incidents to extract recurring failure patterns
3. Generates production-quality operational runbooks in markdown
4. Reviews runbooks for completeness and accuracy
5. Publishes approved runbooks to the GitHub repository wiki
6. Generates a coverage report showing which services have runbook coverage

## Key Files

| File | Purpose |
|---|---|
| `config/settings.yaml` | **Start here** — set `target_repo` to your GitHub repo |
| `data/raw-incidents.json` | Raw incident data fetched from GitHub (auto-generated) |
| `data/incident-analysis.json` | Structured pattern analysis output (auto-generated) |
| `data/runbook-queue.json` | Prioritized list of runbooks to generate (auto-generated) |
| `data/review-results.json` | Quality review results and verdict (auto-generated) |
| `output/runbooks/*.md` | Generated runbooks (the primary output) |
| `output/coverage-report.md` | Human-readable coverage summary |
| `output/coverage-summary.json` | Machine-readable coverage data |
| `templates/runbook-template.md` | Formatting template — procedure-writer uses this |
| `sample-data/sample-incidents.json` | Demo incidents for testing without a real GitHub repo |

## Agent Responsibilities

**incident-analyst** runs two phases:
- `analyze-patterns`: parses raw incidents, writes `data/incident-analysis.json`
- `prioritize-runbooks`: decides which runbooks to generate, writes `data/runbook-queue.json`

**procedure-writer** runs `write-runbooks`:
- Reads `data/runbook-queue.json` and `data/incident-analysis.json`
- Reads `templates/runbook-template.md` for formatting
- Writes `output/runbooks/<service>-<pattern-slug>.md` for each runbook

**runbook-reviewer** runs `review-runbooks`:
- Reviews all files in `output/runbooks/`
- Writes verdict to `data/review-results.json`
- Can send workflow back to write-runbooks with specific feedback (up to 3 times)

**wiki-publisher** runs two phases:
- `publish-to-wiki`: clones wiki repo, copies runbooks, updates index, pushes
- `coverage-report`: writes `output/coverage-report.md` and `output/coverage-summary.json`

## Data Flow

```
GitHub Issues
     │
     ▼ (gh issue list)
data/raw-incidents.json
     │
     ▼ (incident-analyst)
data/incident-analysis.json
     │
     ▼ (incident-analyst)
data/runbook-queue.json
     │
     ▼ (procedure-writer)
output/runbooks/*.md
     │
     ▼ (runbook-reviewer — may loop back)
data/review-results.json
     │
     ▼ (wiki-publisher)
GitHub Wiki pages
     │
     ▼ (wiki-publisher)
output/coverage-report.md
output/coverage-summary.json
```

## Running Without a Real GitHub Repo

If you don't have a GitHub repo with incident-labeled issues, use the sample data:

```bash
cp sample-data/sample-incidents.json data/raw-incidents.json
```

Then trigger the workflow — the fetch-incidents phase will produce an empty file if no
issues are found, and analyze-patterns will fall back to `sample-data/sample-incidents.json`
automatically.

## Configuration Reference (config/settings.yaml)

```yaml
target_repo: "myorg/myapp"         # GitHub repo to fetch incidents from
incident_labels:                    # Labels that identify incident issues
  - incident
  - outage
  - postmortem
  - sev1
  - sev2
severity_mapping:                   # Map GitHub labels to P1/P2/P3/P4
  sev1: P1
  outage: P1
  sev2: P2
  incident: P2
  postmortem: P2
services:                           # Known services (for coverage tracking)
  - api-gateway
  - auth-service
  - database
coverage_threshold: 80              # Minimum coverage % to pass the check
staleness_days: 30                  # Flag runbooks older than this
```

## Extending This Workflow

**Add a new service:** Update `services` list in `config/settings.yaml`.

**Change incident labels:** Add labels to `incident_labels` in `config/settings.yaml`.

**Add a new agent phase:** Add an agent in `agents.yaml`, a phase in `phases.yaml`,
and insert the phase in the workflow sequence in `workflows.yaml`.

**Adjust the rework limit:** Change `max_rework_attempts` in `workflows.yaml` for the
`review-runbooks` phase.

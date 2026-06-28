---
name: runbook-generator
description: >
  Use this skill whenever a user wants to write, generate, or create an operational runbook
  for a service, application, or infrastructure component. Triggers include: "write a runbook
  for", "generate a runbook", "create ops documentation", "document how to operate X",
  "write a service runbook", "create an on-call guide for", "document deployment and rollback
  for", "write operational procedures for", or when a user describes a service and wants
  structured operational documentation for it. Covers the service types the team builds:
  AI agent services, agent platforms, networking applications (Cisco SD-WAN, monitoring,
  orchestration), Python APIs and scripts. Works from minimal input (service name + description)
  up to rich input (code, configs, architecture notes). Always produces a complete,
  ready-to-use Markdown runbook — not a skeleton or outline.
---

# Runbook Generator Skill

Generates complete **operational runbooks** for the service types this team builds:
AI agents, agent platforms, networking applications (SD-WAN, monitoring, orchestration),
and Python services. Output is Markdown, ready for Confluence, GitHub Wiki, or Notion.

---

## Step 1 — Analyze Input

Identify what the user has provided:

| Input type | What to extract |
|---|---|
| Plain description | Service name, purpose, tech stack, dependencies |
| Code (Python) | Entry point, external calls, config loading, endpoints |
| Config files (env, YAML, TOML) | Required env vars, external services, feature flags |
| Architecture notes | Data flow, upstream/downstream dependencies |
| Existing partial runbook | Gaps to fill, sections to expand |

**Minimum required** to generate a useful runbook:
- Service name and what it does
- Primary tech stack or runtime
- At least one of: code snippet, config, or architecture description

If critical information is missing, ask **one focused question** before proceeding.
Otherwise proceed — make sensible assumptions and list them in an Assumptions section.

---

## Step 2 — Detect Service Type(s)

A service can be multiple types. Identify all that apply — each adds specific supplement
sections to the base runbook.

| Type | Indicators |
|---|---|
| **AI Agent / Agentic service** | LLM API calls, tool use, MCP server, agent orchestration, context management, Anthropic/OpenAI SDK |
| **Agent platform** | Orchestrates multiple agents, manages agent lifecycle, message routing, state persistence across agents |
| **Networking application** | Cisco API integrations (SD-WAN / vManage, ACI, ISE, Nexus, Intersight, Hypershield, SSE), device management, network policy, topology, monitoring |
| **Network monitoring agent** | Polls device telemetry, collects metrics, detects anomalies, generates alerts from network state |
| **Network orchestration agent** | Executes configuration changes, provisions services, automates remediation across network devices |
| **Python API service** | FastAPI/Flask, REST/HTTP endpoints, no agent/LLM logic |
| **Background worker / script** | No HTTP endpoints, scheduled task, event-driven processor, CLI tool |

---

## Step 3 — Build the Runbook

Every runbook starts with the **Base Sections**, then appends the **Type Supplement**
for each detected type.

Read `references/runbook-template.md` for the full base template.
Read `references/supplements.md` for the type-specific sections.

### Base Sections (always include, every service type)

1. **Overview** — what it does, who owns it, criticality level (P1/P2/P3)
2. **Architecture & Dependencies** — what it calls, what calls it, what breaks if it's down
3. **Deployment & Rollback** — how to deploy, how to roll back, how to verify
4. **Health Verification** — how to confirm the service is running correctly
5. **Common Operations** — restart, view logs, check config
6. **Troubleshooting** — symptom → diagnosis → fix for the most likely failure modes
7. **Escalation** — who to call, when, what info to collect first
8. **Assumptions** — list any assumptions made; remove once placeholders are filled

### Type Supplements (add when detected)

| Detected type | Supplement sections to add |
|---|---|
| AI Agent / Agentic service | Model configuration, tool/MCP registration, context & memory, rate limits & quotas, agent-specific troubleshooting |
| Agent platform | Agent lifecycle management, message routing, state store operations, platform-level troubleshooting |
| Networking application | Cisco platform integrations table, API auth management, device connectivity checks, policy verification |
| Network monitoring agent | Telemetry collection health, polling schedule, alert pipeline, data source connectivity |
| Network orchestration agent | Change execution safety, rollback of network changes, idempotency verification, device state validation |
| Background worker / script | Job execution, schedule verification, dead-letter / failure handling |

---

## Step 4 — Writing Guidelines

### Overview
- One paragraph: what the service does, who depends on it, its criticality
- **P1** — outage has direct customer or network operations impact
- **P2** — outage degrades visibility or slows operations but network still functions
- **P3** — non-critical, background, tolerable downtime
- Include: owner team, on-call contact placeholder, last updated date placeholder

### Architecture & Dependencies
- List upstream (what this service calls) and downstream (what calls this service)
- For each dependency: name, type, and what breaks if it's unavailable
- For AI agent services: include LLM provider endpoint and model as a named dependency
- For networking apps: one row per Cisco platform (vManage URL, ISE node, ACI controller, etc.)
- Keep it as a table — easy to scan during an incident

### Deployment & Rollback
- Real commands, not pseudocode
- Always include: pre-deploy check, deploy command, post-deploy verification, rollback command
- For AI services: note if a model version or prompt version is part of the deployment artifact
- For networking apps: note if device configuration is pushed as part of deploy and how to revert it
- For orchestration agents: include a dry-run step before any live execution

### Health Verification
- Every check must be a runnable command or URL with expected output
- Never write "check the logs" without specifying what to look for
- For AI services: include a test inference call to confirm model endpoint is reachable
- For networking apps: include a live API call to at least one Cisco platform to confirm connectivity

### Troubleshooting
Structure every issue as: **Symptom → Diagnosis → Fix → Verify**

```
#### <Symptom>
**Diagnosis:** <commands to confirm root cause>
**Fix:** <commands to resolve>
**Verify:** <commands to confirm recovery>
```

Always cover the most likely failure modes for each detected type:
- AI Agent: model API unreachable, tool call failure, context overflow, rate limit hit
- Networking app / monitoring agent: Cisco controller unreachable, auth token expired, device offline, polling gap, missing metrics
- Orchestration agent: change execution failed mid-flight, device left in partial config state, rollback needed
- Worker/script: job not running on schedule, processing stalled, queue backlog

### Escalation
- T1: What on-call tries first (this runbook)
- T2: When and who to escalate to (team + channel placeholder)
- T3: External vendor — Cisco TAC for platform issues, LLM provider support for model issues
- Always list what diagnostic information to collect before escalating

---

## Step 5 — Output Format

Produce the runbook as a single Markdown document:

```
# Runbook: <Service Name>

| Field | Value |
|---|---|
| **Owner** | <team> |
| **Criticality** | P1 / P2 / P3 |
| **Service Type** | <detected types> |
| **On-call Channel** | #<slack-channel> |
| **Last Updated** | YYYY-MM-DD |

---
<base sections>
<type supplement sections>
```

After the runbook body, add a **"What to fill in"** checklist — every placeholder
the team needs to replace before the runbook is usable.

---

## Edge Cases

- **Vague input**: Generate from what's available; list assumptions; ask one clarifying question at the end
- **Multiple service types**: Include all relevant supplements; group related sections together
- **Pure Python script**: Base sections only — process management, log file locations, manual trigger steps
- **Stateful service**: Add explicit data safety warnings before any destructive operation step
- **Agent with multiple tools/MCPs**: Generate a Tool Registry section listing each tool, its purpose, and its failure impact
- **Networking app with multiple Cisco platforms**: One dependency row per platform; note which operations are blocked per platform outage
- **Orchestration agent**: Always include a "Safety & Rollback" subsection — what partial state looks like and how to clean it up
- **Monitoring agent with polling gap**: Include how to detect a missed poll window and how to backfill or trigger a manual poll

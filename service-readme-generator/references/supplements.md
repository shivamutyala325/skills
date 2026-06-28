# README Supplement Sections

Type-specific sections to insert into the README when a service type is detected.
Place these after the Usage section and before Project Structure unless noted otherwise.

---

## Supplement: AI Agent / Agentic Service

---

### Tools & MCP Servers

List every tool and MCP server the agent uses. This tells a developer what external
services need to be running or configured before the agent can function.

| Tool / MCP Server | Purpose | Setup Required |
|---|---|---|
| `<tool-name>` | <what the agent uses it for> | <how to set it up — e.g., "Set `TOOL_API_KEY` in .env"> |
| `<mcp-server-name>` | <what it provides> | <setup instructions or link> |

### Example Interactions

Show 2–3 representative interactions that demonstrate what the agent can do.
Use realistic inputs and outputs — not toy "hello world" examples.

**Example 1: <What this shows>**

Input:
```
<user message or trigger input>
```

Agent response:
```
<realistic agent output>
```

**Example 2: <What this shows>**

Input:
```
<user message>
```

Agent response:
```
<realistic output showing tool use or multi-step reasoning>
```

### Extending with New Tools

How a developer adds a new tool to this agent:

```python
# 1. Define the tool function in tools/<tool_name>.py
def my_new_tool(param: str) -> dict:
    """<Tool description — this becomes the tool's description for the LLM>"""
    ...

# 2. Register it in agent.py (or wherever tools are registered)
from tools.my_new_tool import my_new_tool

tools = [
    ...,
    my_new_tool,
]
```

<Any other steps specific to this agent's tool registration pattern.>

---

## Supplement: Agent Platform

---

### Supported Agent Types

List the agent types this platform can run:

| Agent Type | Description | Entry Point |
|---|---|---|
| `<agent-type-name>` | <what it does> | `<module or command to launch it>` |

### Registering an Agent

How to register a new agent with the platform:

```python
# <code example showing how to define and register a new agent>
```

Or via config:
```yaml
# <config file example showing agent registration>
```

### Platform API Reference

If the platform exposes an HTTP API for managing agents:

#### `POST /agents` — Start an Agent Session

Request:
```json
{
  "<field>": "<value>",
  "<field>": "<value>"
}
```

Response:
```json
{
  "session_id": "<uuid>",
  "status": "running"
}
```

#### `GET /agents/{session_id}` — Get Session Status

Response:
```json
{
  "session_id": "<uuid>",
  "status": "running | completed | failed",
  "result": "<output if completed>"
}
```

#### `DELETE /agents/{session_id}` — Stop a Session

Response: `204 No Content`

---

## Supplement: Networking Application

---

### Cisco Platform Requirements

This service integrates with the following Cisco platforms.
Ensure you have credentials and network access to each before running.

| Platform | Required | Env Variable(s) | Notes |
|---|---|---|---|
| SD-WAN (vManage) | Yes / No | `VMANAGE_HOST`, `VMANAGE_USER`, `VMANAGE_PASSWORD` | vManage 20.x+ required |
| ACI (APIC) | Yes / No | `APIC_HOST`, `APIC_USER`, `APIC_PASSWORD` | |
| ISE | Yes / No | `ISE_HOST`, `ISE_USER`, `ISE_PASSWORD` | ERS API must be enabled |
| Nexus Dashboard | Yes / No | `ND_HOST`, `ND_API_KEY` | |
| Intersight | Yes / No | `INTERSIGHT_API_KEY_ID`, `INTERSIGHT_API_KEY_SECRET` | |
| Hypershield | Yes / No | `<env vars>` | |
| SSE | Yes / No | `<env vars>` | |

### Credential Setup

```bash
# SD-WAN / vManage
VMANAGE_HOST=https://your-vmanage-host
VMANAGE_USER=your-username
VMANAGE_PASSWORD=your-password

# Add other platforms as needed
```

> Credentials are loaded from environment variables. Never commit `.env` to version control.

### Supported Operations

What this service can do against the configured platforms:

| Operation | Platform | Description |
|---|---|---|
| `<operation-name>` | `<platform>` | <what it does> |
| `<operation-name>` | `<platform>` | <what it does> |

### Example: <Most Common Operation>

```python
# <code or CLI example showing the most common operation>
```

Or via API:
```bash
curl -X POST http://localhost:8000/v1/<endpoint> \
  -H "Content-Type: application/json" \
  -d '{"<field>": "<value>"}'
```

Expected response:
```json
{
  "<field>": "<value>"
}
```

---

## Supplement: Network Monitoring Agent

---

### What This Agent Monitors

| Metric / Data Type | Source | Collection Method | Frequency |
|---|---|---|---|
| `<metric name>` | `<device type or platform>` | <SNMP / REST API / streaming telemetry> | Every `<N>` seconds |
| `<metric name>` | `<source>` | <method> | `<interval>` |

### Output & Data Storage

Where collected metrics are written:

| Destination | Format | Connection Config |
|---|---|---|
| `<InfluxDB / Prometheus / Elasticsearch / file>` | `<line protocol / JSON / etc.>` | `<env variable>` |

### Alert Configuration

How to configure alerts when thresholds are breached:

```yaml
# alerts.yaml (or equivalent config)
alerts:
  - name: <alert-name>
    metric: <metric-name>
    condition: <threshold condition>
    destination: <slack-webhook / pagerduty / etc.>
```

Or via environment variables:
```bash
ALERT_THRESHOLD_<METRIC>=<value>
ALERT_WEBHOOK_URL=<your-webhook-url>
```

### Viewing Collected Metrics

```bash
# Query latest data for all devices
<command to query the data store>

# Query a specific device
<command with device filter>

# Example output:
# <what the data looks like>
```

### Adding a New Device to Monitor

```bash
# Option 1: via config file
# Add to devices.yaml:
# - host: <device-ip>
#   type: <device-type>
#   credentials: <credential-profile>

# Option 2: via API (if supported)
curl -X POST http://localhost:8000/v1/devices \
  -H "Content-Type: application/json" \
  -d '{"host": "<device-ip>", "type": "<device-type>"}'
```

---

## Supplement: Network Orchestration Agent

---

### Supported Change Types

What this agent can configure or provision:

| Change Type | Target Platform | Description | Reversible |
|---|---|---|---|
| `<change-type>` | `<platform>` | <what it does> | Yes / No |
| `<change-type>` | `<platform>` | <what it does> | Yes / No |

### Safety Model

This agent operates in two modes:

| Mode | Behavior | How to Activate |
|---|---|---|
| **Dry-run** (default) | Generates a change plan — no changes applied | Default; set `DRY_RUN=true` |
| **Live** | Applies changes to devices | Set `DRY_RUN=false` — use with care |

> **Always run in dry-run mode first.** Review the plan before switching to live execution.

### Dry-Run Example

```bash
# Preview what would change — no changes applied
DRY_RUN=true python main.py --intent "<describe what you want>"

# Example output:
# [DRY RUN] Plan:
#   MODIFY device: <device-name>
#     - set interface GigabitEthernet0/0 description "uplink-to-core"
#     - add bgp neighbor 10.0.0.1 remote-as 65001
# No changes applied.
```

### Live Execution Example

```bash
# Apply the change
DRY_RUN=false python main.py --intent "<describe what you want>"

# Example output:
# [LIVE] Applying changes to <device-name>...
# ✓ interface description updated
# ✓ bgp neighbor added
# Change ID: <uuid>  ← save this for rollback
```

### Rolling Back a Change

```bash
# Roll back by change ID
python main.py --rollback <change-id>

# Expected output:
# Rolling back change <change-id>...
# ✓ Reverted: interface description
# ✓ Reverted: bgp neighbor removed
```

### Audit Log

Every live execution is logged. To view recent changes:

```bash
<command to view audit log>
```

---

## Supplement: Python API Service

---

### API Reference

Base URL: `http://localhost:<port>/v1`

Authentication: `Authorization: Bearer <token>`  (or document the actual auth method)

#### `GET /<resource>` — List Resources

Query parameters:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `cursor` | string | No | Pagination cursor from previous response |
| `limit` | integer | No | Items per page (default: 20, max: 100) |

Response:
```json
{
  "data": [
    {
      "<field>": "<value>"
    }
  ],
  "pagination": {
    "hasMore": true,
    "nextCursor": "<cursor-string>"
  }
}
```

#### `POST /<resource>` — Create Resource

Request body:
```json
{
  "<required-field>": "<value>",
  "<optional-field>": "<value>"
}
```

Response: `201 Created`
```json
{
  "id": "<uuid>",
  "<field>": "<value>"
}
```

#### `GET /<resource>/{id}` — Get Resource

Response: `200 OK`
```json
{
  "id": "<uuid>",
  "<field>": "<value>"
}
```

#### `DELETE /<resource>/{id}` — Delete Resource

Response: `204 No Content`

#### Error Responses

All errors follow RFC 7807 Problem Details format:
```json
{
  "type": "https://api.example.com/errors/<error-type>",
  "title": "<short description>",
  "status": 400,
  "detail": "<specific error message>"
}
```

---

## Supplement: CLI Tool / Script

---

### Command Reference

```
<tool-name> [OPTIONS] COMMAND [ARGS]
```

### Commands

#### `<command-name>` — <what it does>

```bash
<tool-name> <command-name> [OPTIONS]
```

Options:

| Flag | Short | Required | Default | Description |
|---|---|---|---|---|
| `--<option>` | `-<short>` | Yes / No | `<default>` | <description> |

Example:
```bash
<tool-name> <command-name> --<option> <value>

# Output:
# <expected output>
```

#### `<command-name>` — <what it does>

```bash
<tool-name> <command-name> --<option> <value>

# Output:
# <expected output>
```

### Global Options

| Flag | Description |
|---|---|
| `--help` | Show help and exit |
| `--version` | Show version and exit |
| `--dry-run` | Preview actions without executing (if applicable) |
| `--verbose` / `-v` | Enable verbose logging |

# Runbook Supplement Sections

Type-specific sections to append to the base runbook when a service type is detected.
Include only the supplements that match the service being documented.

---

## Supplement: AI Agent / Agentic Service

Add when the service uses LLM APIs, tool use, MCP servers, or agent orchestration.

---

### Model & Tool Configuration

| Field | Value |
|---|---|
| **LLM Provider** | Anthropic / OpenAI / etc. |
| **Model** | `<model-id>` (e.g., `claude-sonnet-4-6`) |
| **Model Version Pin** | Yes / No — pinned to `<version>` |
| **Context Window** | `<N>` tokens |
| **Max Output Tokens** | `<N>` |
| **Temperature** | `<value>` |
| **Tools / MCP Servers** | See Tool Registry below |

### Tool Registry

| Tool / MCP Server | Purpose | Required | Failure Impact |
|---|---|---|---|
| `<tool-name>` | <what the agent uses it for> | Yes / No | <what breaks if unavailable> |
| `<mcp-server-name>` | <what it provides> | Yes / No | <what breaks if unavailable> |

### Agent Health Verification

```bash
# Confirm LLM API is reachable
curl -s -o /dev/null -w "%{http_code}" \
  https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-haiku-4-5-20251001","max_tokens":10,"messages":[{"role":"user","content":"ping"}]}'
# Expected: 200

# Confirm MCP servers are registered
<command to list or ping registered MCP servers>

# Send a minimal test inference
<command for a test agent run>
# Expected: <what a successful response looks like>
```

### Agent Common Operations

#### Check Rate Limit / Quota Status
**When:** Slow responses or 429 errors.
```bash
<command or dashboard URL to check current quota usage>
```

#### Clear / Reset Agent Context or Memory
**When:** Agent behaving unexpectedly due to stale or corrupted context.
```bash
<command to clear context store>
```
**Verify:** Send a fresh test message; confirm expected behavior.

#### View Agent Execution Trace
**When:** Diagnosing an unexpected tool call or agent action.
```bash
<command to retrieve execution trace or tool call log>
```

### Agent Troubleshooting

#### LLM API Unreachable / 5xx

**Diagnosis:**
```bash
curl -s -o /dev/null -w "%{http_code}" https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-haiku-4-5-20251001","max_tokens":10,"messages":[{"role":"user","content":"ping"}]}'
```
**Fix:** Check `https://status.anthropic.com`; verify API key is valid.
**Verify:** Test inference returns 200.

---

#### Tool Call Failure / MCP Server Unreachable

**Diagnosis:**
```bash
<command to check MCP server status>
<log command> | grep -i "tool_error\|mcp\|tool call failed"
```
**Fix:** Restart MCP server: `<restart command>`; verify credentials are set.
**Verify:** Test request exercising the tool completes successfully.

---

#### Rate Limit Hit (429)

**Diagnosis:**
```bash
<log command> | grep "429\|rate_limit\|RateLimitError"
```
**Fix:** Reduce concurrency; add backoff; review tier limits with provider.
**Verify:** Requests resume without 429 errors.

---

#### Context Window Overflow

**Diagnosis:**
```bash
<log command> | grep "context_length\|max_tokens\|token limit"
```
**Fix:** Reduce messages in context; add summarization or truncation.
**Verify:** Agent processes requests without token errors.

---

## Supplement: Agent Platform

Add when the service orchestrates multiple agents or manages agent lifecycle.

---

### Platform Configuration

| Field | Value |
|---|---|
| **Orchestration mode** | Sequential / Parallel / Event-driven |
| **State store** | `<Redis / PostgreSQL / in-memory / etc.>` |
| **Message broker** | `<Kafka / RabbitMQ / SQS / none>` |
| **Agent registry** | `<how agents are registered and discovered>` |
| **Max concurrent agents** | `<N>` |

### Platform Health Verification

```bash
# Platform API health
<health check command>

# State store reachable
<command to ping state store>

# Message broker reachable (if applicable)
<command to check broker connectivity>

# Active agent sessions
<command to list running agent sessions>
```

### Platform Common Operations

#### View Active Agent Sessions
```bash
<command to list running agents with status>
```

#### Kill a Stuck Session
```bash
<command to terminate a specific agent session by ID>
```
**Verify:** Session no longer in active list.

#### Drain Before Maintenance
```bash
# Pause new session intake
<command to pause ingress>

# Wait for active sessions to finish
<command to watch session count reach 0>

# Redeploy
<deploy command>
```

### Platform Troubleshooting

#### Agents Not Starting / Stuck in Queue

**Diagnosis:**
```bash
<command to check queue depth>
<command to check executor health>
<log command> | grep -i "queue\|dispatch\|executor"
```
**Fix:** Verify executor processes are running; confirm state store is reachable.

---

#### State Store Inconsistency

**Diagnosis:**
```bash
<command to inspect state for a specific session>
<command to check for orphaned entries>
```
**Fix:** For a single session: `<command to clear/reset state>`. Coordinate with team before clearing shared state.

---

## Supplement: Networking Application

Add when the service integrates with Cisco platforms, manages devices, or handles network policy.

---

### Cisco Platform Integrations

| Platform | Endpoint | Purpose | Auth Method |
|---|---|---|---|
| SD-WAN (vManage) | `https://<vmanage-host>` | <purpose> | Session token |
| ACI Controller (APIC) | `https://<apic-host>` | <purpose> | Cookie token |
| ISE | `https://<ise-host>:9060` | <purpose> | Basic auth / ERS API |
| Nexus Dashboard | `https://<nd-host>` | <purpose> | API key / token |
| Intersight | `https://intersight.com` | <purpose> | API key (v3) |
| Hypershield | `<endpoint>` | <purpose> | `<auth method>` |
| SSE | `<endpoint>` | <purpose> | `<auth method>` |

### Network Integration Health Verification

```bash
# SD-WAN / vManage — confirm reachable and auth works
curl -sk -c /tmp/vmanage-cookie -o /dev/null -w "%{http_code}" \
  -X POST https://<vmanage-host>/j_security_check \
  -d "j_username=<user>&j_password=<pass>"
# Expected: 200

# ACI — login and confirm token
curl -sk -o /dev/null -w "%{http_code}" \
  https://<apic-host>/api/aaaLogin.json \
  -d '{"aaaUser":{"attributes":{"name":"<user>","pwd":"<pass>"}}}'
# Expected: 200

# Confirm device inventory is accessible
<command to list devices or sites from the platform>
```

### Network Common Operations

#### Refresh Authentication Token
**When:** API calls return 401 or token expired errors.
```bash
<command to re-authenticate and refresh token>
```
**Verify:** Next API call returns 200.

#### Check Device Connectivity
**When:** Operations on a specific device are failing.
```bash
<command to query device status via platform API>
<command to check device sync status>
```

#### Re-sync Device State
**When:** Device config is out of sync with controller.
```bash
<command to trigger re-sync or re-discovery>
```
**Verify:** Device shows as synced in API response.

### Network Troubleshooting

#### Cisco Controller Unreachable

**Diagnosis:**
```bash
ping <controller-host>
curl -sk -o /dev/null -w "%{http_code}" https://<controller-host>/...
<log command> | grep "401\|connection refused\|timeout\|unreachable"
```
**Fix:** Check network path to controller; refresh auth token if 401; escalate to platform team if controller is down.
**Verify:** API call returns 200 with valid data.

---

#### Auth Token Expired

**Diagnosis:**
```bash
<log command> | grep "401\|token expired\|session invalid"
```
**Fix:** Re-authenticate: `<auth refresh command>`.
**Verify:** Next API call returns 200.

---

#### API Version Mismatch

**Diagnosis:**
```bash
<log command> | grep -i "version\|deprecated\|unsupported api"
```
**Fix:** Check current platform version: `<version query command>`; update API version in service config.
**Verify:** API calls return expected data without version warnings.

---

#### Device Offline / Not Responding

**Diagnosis:**
```bash
<command to query device health from controller API>
<command to check device last contact timestamp>
```
**Fix:** If genuinely offline: escalate to network operations. If shows online but API fails: check credentials and access policy on the device.

---

## Supplement: Network Monitoring Agent

Add when the service collects telemetry, polls device metrics, detects anomalies, or generates alerts.

---

### Monitoring Configuration

| Field | Value |
|---|---|
| **Poll interval** | `<N>` seconds / minutes |
| **Data sources** | `<list of devices, controllers, or APIs being polled>` |
| **Metrics collected** | `<list of metric types: interface stats, BGP state, CPU, etc.>` |
| **Alert destinations** | `<Slack channel / PagerDuty / webhook / etc.>` |
| **Data store** | `<where metrics are written: InfluxDB / Prometheus / Elasticsearch / etc.>` |
| **Retention** | `<how long metrics are kept>` |

### Monitoring Health Verification

```bash
# Confirm the agent is running and polling
<command to check process status or last poll timestamp>

# Confirm data is flowing into the data store
<command to query latest data point — should be within 1 poll interval>

# Confirm all configured devices are reachable
<command to list device poll status>
# Expected: all devices show last_seen within <poll interval>

# Confirm alerts are firing correctly (use a known test device or condition)
<command to trigger a test alert or check last alert sent>
```

### Monitoring Common Operations

#### Trigger a Manual Poll
**When:** Verifying a fix, checking a device immediately without waiting for the schedule.
```bash
<command to trigger an on-demand poll for a specific device or all devices>
```
**Verify:** New data point appears in the data store within a few seconds.

#### Check Polling Gap (Missed Polls)
**When:** Alerts fired about missing data, or dashboard shows gaps.
```bash
# Find devices with no recent data
<command to query data store for devices with no data in the last N minutes>

# Check poll error log
<log command> | grep -i "poll failed\|timeout\|unreachable"
```

#### Add / Remove a Device from Monitoring
```bash
<command to add a device to the monitoring config>
<command to remove a device>
```
**Verify:** Device appears/disappears in next poll cycle.

#### Silence an Alert
**When:** Known maintenance window; suppressing a noisy alert.
```bash
<command to silence or suppress a specific alert>
```
**Verify:** Alert not re-fired during maintenance window.

### Monitoring Troubleshooting

#### Polling Gap — No Data from One or More Devices

**Diagnosis:**
```bash
<log command> | grep -i "poll failed\|connection refused\|timeout" | grep "<device-ip or name>"
<command to check device reachability>
```
**Fix:**
- Device unreachable: check network path; escalate to network ops if persistent
- Auth failure: refresh credentials for that device
- Agent crashed: restart agent: `<restart command>`

**Verify:** New data point for the device appears in data store.

---

#### Alert Not Firing When Expected

**Diagnosis:**
```bash
# Confirm the condition is actually present
<command to check raw metric value that should have triggered alert>

# Check alert rule configuration
<command to inspect alert rule threshold and condition>

# Check alert delivery log
<log command> | grep -i "alert\|notification\|webhook"
```
**Fix:**
- Threshold too high: adjust alert rule
- Notification destination down: check Slack/PagerDuty integration
- Polling gap: resolve data collection issue first

---

#### False Positive Alerts — Alert Firing Incorrectly

**Diagnosis:**
```bash
# Verify the raw metric vs the alert condition
<command to query raw metric around the time the alert fired>
```
**Fix:** Adjust threshold or add a minimum duration requirement to the alert rule.

---

#### Data Store Falling Behind / High Write Latency

**Diagnosis:**
```bash
<command to check data store write latency or queue depth>
<command to check disk/memory on data store host>
```
**Fix:** Scale the data store; reduce poll frequency temporarily; check for expensive queries running against it.

---

## Supplement: Network Orchestration Agent

Add when the service executes configuration changes, provisions services, or automates remediation across network devices.

---

### Orchestration Configuration

| Field | Value |
|---|---|
| **Execution mode** | Dry-run by default / Live by default |
| **Target platforms** | `<SD-WAN / ACI / ISE / etc.>` |
| **Change scope** | `<what types of changes this agent can make>` |
| **Rollback strategy** | Automatic / Manual / Not supported |
| **Idempotent** | Yes / No |
| **Approval required** | Yes (via `<approval mechanism>`) / No |
| **Audit log** | `<where changes are logged>` |

### Orchestration Health Verification

```bash
# Confirm agent is running and connected to target platforms
<command to check agent health and platform connectivity>

# Confirm dry-run mode works (safe test)
<command to run agent in dry-run mode against a test target>
# Expected: plan output with no actual changes applied

# Confirm audit log is being written
<command to check last N entries in audit log>
```

### Orchestration Common Operations

#### Run in Dry-Run Mode (Preview Changes)
**When:** Before any live execution, or during testing.
```bash
<command to run the orchestration agent in dry-run / plan mode>
# Expected: output shows what WOULD change, nothing is applied
```

#### Execute a Change (Live)
```bash
# 1. Dry-run first
<dry-run command>

# 2. Review the plan output

# 3. Execute if plan looks correct
<live execution command>

# 4. Verify
<verification command>
```

#### Roll Back a Change
**When:** A change caused unexpected behavior or needs to be reverted.
```bash
# Check change audit log for the change ID / snapshot
<command to list recent changes with IDs>

# Roll back a specific change
<rollback command with change ID>
```
**Verify:** Device config matches pre-change state: `<command to compare current vs expected state>`.

#### View Change Audit Log
```bash
<command to view recent changes: timestamp, target, what changed, who triggered>
```

### Orchestration Troubleshooting

#### Change Failed Mid-Execution — Partial State

**Diagnosis:**
```bash
# Find the failed change in audit log
<log command> | grep -i "failed\|partial\|rollback"

# Check current device state vs expected
<command to diff current device config against intended config>
```
**Fix:**
- If agent supports auto-rollback: `<rollback command>`
- If manual: identify which steps completed, manually revert only those
- Always validate final device state before declaring resolved

**Verify:** `<command to confirm device is in expected clean state>`.

---

#### Agent Applying Wrong Change / Unintended Target

**Diagnosis:**
```bash
# Check what the agent received as input
<log command> | grep -i "target\|scope\|input" | tail -20

# Check if the dry-run was reviewed before execution
<audit log command for the specific execution>
```
**Fix:** Halt any in-flight executions; roll back the applied change; investigate input source (misconfigured intent, wrong device selector).

---

#### Rollback Failed

**Diagnosis:**
```bash
<log command> | grep -i "rollback failed\|cannot revert"
<command to inspect current device state>
```
**Fix:**
- Manual revert: retrieve pre-change config snapshot from audit log and apply it directly via the platform UI or API
- Escalate to network operations team with the audit log entry and current device state

---

## Supplement: Background Worker / Script

Add when the service has no HTTP endpoints and runs as a scheduled job, event-driven processor, or CLI tool.

---

### Job Configuration

| Field | Value |
|---|---|
| **Schedule** | `<cron expression>` or "event-triggered" |
| **Entry point** | `<command to run>` |
| **Expected duration** | `<typical runtime>` |
| **Idempotent** | Yes / No |
| **Input source** | `<queue / file path / API poll>` |
| **Output / side effects** | `<what it writes or changes>` |

### Worker Health Verification

```bash
# Confirm last run completed successfully
<command to check last execution log or job history>

# Check for queue backlog (queue-based workers)
<command to check queue depth>
```

### Worker Common Operations

#### Trigger a Manual Run
```bash
<command to run the script/job manually>
# Expected: exits 0, output shows <expected summary>
```

#### Check Job History
```bash
<command to list recent runs with status and duration>
```

#### Clear a Stalled Queue
```bash
<command to check dead-letter queue or unacked messages>
<command to requeue or discard as appropriate>
```

### Worker Troubleshooting

#### Job Not Running on Schedule

**Diagnosis:**
```bash
<command to check scheduler / cron status>
<command to check errors in last scheduled run>
```
**Fix:** Verify schedule expression; check process hasn't crashed; check for a concurrency lock preventing re-entry.

---

#### Processing Stalled Mid-Run

**Diagnosis:**
```bash
<command to confirm process is still running>
<log command> | tail -20
```
**Fix:** If waiting on a dependency: check that dependency. If stuck in a loop: kill and restart; investigate the input that triggered the stall.

# Base Runbook Template

Generic template that applies to every service type.
Type-specific supplement sections are in `supplements.md`.

---

# Runbook: <Service Name>

| Field | Value |
|---|---|
| **Owner** | <Team Name> |
| **Criticality** | P1 / P2 / P3 |
| **Service Type** | <AI Agent / Agent Platform / Networking App / Monitoring Agent / Orchestration Agent / Worker> |
| **Repository** | `<github-org>/<repo-name>` |
| **On-call Channel** | `#<slack-channel>` |
| **Escalation Contact** | `<team-lead or on-call rotation>` |
| **Last Updated** | YYYY-MM-DD |

---

## Overview

<Service Name> is a <service type> responsible for <one sentence: what it does and why it matters>.
It is consumed by <downstream services or users> and depends on <upstream services or platforms>.

**Criticality:** This is a **P<N>** service. An outage causes <specific business or operational impact>.

---

## Architecture & Dependencies

### What This Service Calls (Upstream)

| Dependency | Type | What Breaks if Unavailable |
|---|---|---|
| `<service or API name>` | <LLM API / REST API / Database / Queue / Cisco Platform> | <specific failure mode> |
| `<service or API name>` | <type> | <specific failure mode> |

### What Calls This Service (Downstream)

| Dependent | How It Uses This Service | Impact of Outage |
|---|---|---|
| `<consumer name>` | <how it uses this service> | <specific impact> |

### Data Flow

```
<Brief ASCII or text description of data flow>
Example:
[User / Caller] → [<Service Name>] → [<Dependency 1>]
                                   → [<Dependency 2>]
```

---

## Deployment & Rollback

### Prerequisites
- <List access, credentials, tools, or environment variables needed before deploying>
- Example: Python 3.11+, `ANTHROPIC_API_KEY` set, target environment confirmed

### Deploy

```bash
# <Step 1: pre-deploy check>

# <Step 2: deploy command>

# <Step 3: post-deploy verification>
```

### Verify Deployment Succeeded

```bash
# <Command to confirm the service is running>
# Expected: <what a successful response looks like>

# <Command to check logs for startup errors>
# Expected: <what clean startup logs look like>
```

### Rollback

```bash
# <Command to roll back to previous version>

# <Command to verify rollback succeeded>
```

> **What rollback restores:** <what state it returns to>
> **What rollback does NOT restore:** <migrations, pushed configs, external state changes>

---

## Health Verification

### Health Checks

```bash
# <Health endpoint or process check with expected output>
# Example:
curl -s http://localhost:8000/healthz
# Expected: HTTP 200, {"status": "ok"}

# <Secondary check — dependency connectivity>
# Expected: <what a healthy response looks like>
```

### Confirm the Service is Functional

```bash
# <A lightweight end-to-end check — not just "is it running" but "does it work">
# Example for an agent: send a test message and confirm a response
# Example for a networking app: query a device and confirm a result
# Example for a worker: trigger a job and confirm it completes
```

### Monitoring

- **Dashboard:** `<URL>` — key panels: <error rate, latency, queue depth, etc.>
- **Alerts:** `#<alerts-channel>` — alert names: `<ServiceHighErrorRate>`, `<ServiceDown>`
- **Logs:** `<log location or command>` — look for: `ERROR`, `CRITICAL`, `connection refused`

---

## Common Operations

#### Restart the Service
**When:** Service appears stuck, config change not picked up, memory growth.
```bash
# <restart command>
```
**Verify:** `<command to confirm it restarted cleanly>`

---

#### View Live Logs
```bash
# <command to tail logs>
```

---

#### Check Current Configuration
```bash
# <command to inspect active config / env vars>
```

---

#### Trigger a Manual Run / Test Request
**When:** Confirming the service processes correctly after a deploy or fix.
```bash
# <command or curl to send a test request or trigger a manual job run>
# Expected: <what a successful response looks like>
```

---

## Troubleshooting

#### <Symptom 1 — most common failure mode>

**Diagnosis:**
```bash
# <commands to identify the root cause>
```

**Fix:**
```bash
# <commands to resolve>
```

**Verify:**
```bash
# <commands to confirm recovery>
```

---

#### <Symptom 2>

**Diagnosis:**
```bash
# <commands>
```

**Fix:**
```bash
# <commands>
```

**Verify:**
```bash
# <commands>
```

---

## Escalation

### Before Escalating — Collect This

```bash
# <commands to gather diagnostic info to hand off>
# Minimum: logs, health check output, last known working state
```

### Escalation Path

| Tier | Who | When | How |
|---|---|---|---|
| **T1** | On-call engineer | First response — follow this runbook | `#<oncall-channel>` |
| **T2** | <Service Owner / Team> | After 15 min, or data loss / security risk | `#<team-channel>` / PagerDuty |
| **T3** | <Vendor / Platform team> | Platform-level failure (LLM provider, Cisco TAC, cloud provider) | `<support URL or contact>` |

---

## Assumptions

> *(Remove this section once all placeholders are replaced)*

- <Assumption 1 — e.g., service runs as a systemd unit; update if containerized>
- <Assumption 2 — e.g., health endpoint is /healthz; verify against actual implementation>
- <Assumption 3>

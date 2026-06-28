---
name: service-readme-generator
description: >
  Use this skill whenever a user wants to write, generate, or improve a README for a service,
  application, agent, or Python module. Triggers include: "write a README for", "generate a
  README", "create documentation for this service", "document this agent", "write setup docs",
  "create a README from this code", "document this project", "write developer docs for", or
  when a user pastes code or a project structure and wants documentation. Covers all service
  types the team builds: AI agent services, agent platforms, networking applications (SD-WAN,
  Cisco integrations), network monitoring agents, network orchestration agents, Python APIs
  and scripts. Works from minimal input (service name + description) up to rich input (code,
  config files, project structure). Always produces a complete, developer-ready Markdown README
  — not a skeleton or outline. A README is developer-facing (setup, usage, development) —
  not an ops runbook. Use runbook-generator for operational runbooks.
---

# Service README Generator Skill

Generates **complete, developer-ready README files** for the service types this team builds.
Output is Markdown. A README gets a developer from zero to running in under 5 minutes —
it is not an ops runbook, architecture doc, or API contract.

---

## Step 1 — Analyze Input

Identify what the user has provided:

| Input type | What to extract |
|---|---|
| Plain description | Service name, purpose, tech stack, primary use case |
| Python code | Entry points, endpoints, external calls, CLI args, config loading |
| Config / env file | Required env vars, external service names, feature flags |
| Project directory structure | Module layout, key files, test structure |
| `requirements.txt` / `pyproject.toml` | Dependencies, Python version, installed CLI commands |
| Existing partial README | Gaps to fill, sections to improve |

**Minimum required** to generate a useful README:
- Service name and what it does
- Primary tech stack or runtime
- At least one of: code, config file, or architecture description

If critical information is missing, ask **one focused question** before proceeding.
Otherwise make sensible assumptions and list them at the end.

---

## Step 2 — Detect Service Type(s)

Identify all types that apply — each adds supplement sections to the base README.

| Type | Indicators |
|---|---|
| **AI Agent / Agentic service** | LLM API calls, tool use, MCP server, agent orchestration, Anthropic/OpenAI SDK |
| **Agent platform** | Orchestrates multiple agents, manages agent lifecycle, message routing between agents |
| **Networking application** | Cisco API integrations (SD-WAN/vManage, ACI, ISE, Nexus, Intersight), device management, network policy |
| **Network monitoring agent** | Polls device telemetry, collects metrics, generates alerts from network state |
| **Network orchestration agent** | Executes config changes, provisions services, automates remediation on network devices |
| **Python API service** | FastAPI/Flask, REST endpoints, no agent/LLM logic |
| **CLI tool / script** | `argparse` / `typer`, invoked from the command line, no persistent HTTP server |

---

## Step 3 — Build the README

Every README starts with the **Base Sections**, then adds **Type Supplements** for each detected type.

Read `references/readme-template.md` for the full base template.
Read `references/supplements.md` for type-specific sections.

### Base Sections (always include)

1. **Header** — service name, one-line description, status badges (if repo info available)
2. **Overview** — 2–4 sentences: what it does, why it exists, who uses it
3. **Quick Start** — get from zero to running in under 5 minutes
4. **Configuration** — every environment variable, required vs optional, safe defaults
5. **Usage** — how to actually use the service with real examples
6. **Project Structure** — key files and what they do (skip boilerplate like `__init__.py`)
7. **Development** — local setup, running tests, linting, contributing
8. **License** — one line if known; omit if not provided

### Type Supplements (add when detected)

| Detected type | Supplement sections |
|---|---|
| AI Agent / Agentic service | Tools & MCP servers, example agent interactions, extending with new tools |
| Agent platform | Supported agent types, agent registration, platform API reference |
| Networking application | Cisco platform requirements, credential setup, supported operations |
| Network monitoring agent | What metrics are collected, output format, data store setup, alert configuration |
| Network orchestration agent | Supported change types, dry-run guide, rollback behavior, safety model |
| Python API service | API reference (endpoints, request/response shapes) |
| CLI tool / script | Command reference with all flags and examples |

---

## Step 4 — Writing Guidelines

### Header
```markdown
# <Service Name>

> <One sentence: what it does and the primary outcome it delivers.>
```
Add badges only if repo/CI information is provided — never invent badge URLs.

### Overview
- 2–4 sentences max
- Answer: what does this do, what problem does it solve, and who uses it
- For agents: mention the LLM provider and what the agent is specialized for
- For networking apps: mention which Cisco platforms and what operations it supports
- Do NOT describe the architecture in detail here — that belongs in architecture docs

### Quick Start
- Must work end-to-end in under 5 minutes
- Always starts with: prerequisites → clone/install → configure → run → verify
- Every command must be copy-pasteable — no `<angle bracket>` placeholders in commands
- For verification: show what successful output looks like, not just "it should work"

### Configuration
- Always a table: variable name, required/optional, default, description
- Never omit a variable — if it's in the code, it's in this table
- Group logically: authentication, service endpoints, behavior tuning
- For secrets (API keys, passwords): description says where to get it, not what it looks like
- Include a `.env.example` block showing all variables with placeholder values

### Usage
- Real commands, real inputs, real expected outputs
- Lead with the most common use case — not the most impressive one
- For agents: show an example interaction (input → agent response)
- For networking apps: show an example API call with a real-looking (not sensitive) response
- For CLIs: show the most common command invocations
- Use code blocks for all commands and expected output

### Project Structure
- List only meaningful files — skip standard boilerplate
- One short phrase per entry explaining purpose
- For larger projects: group by subdirectory with a heading per directory

```
service-name/
├── main.py          # entry point
├── config.py        # pydantic-settings config
├── agent.py         # agent loop and tool registration
├── tools/           # MCP tool implementations
│   └── ...
└── tests/
    └── ...
```

### Development
- Step-by-step local setup that actually works
- Include: create venv, install deps, copy `.env.example`, run tests
- Show the test command and what passing output looks like
- Mention linting/formatting tools if present (`ruff`, `black`, `mypy`)

---

## Step 5 — Output Format

Produce a single Markdown document in this order:

```
# <Service Name>

> <tagline>

## Overview
## Quick Start
## Configuration
## Usage
<type-specific supplement sections>
## Project Structure
## Development
## License
```

After the README, add a short **"What to fill in"** note listing any placeholders left
(repository URLs, real example values, badge URLs) — so the developer knows what to
customize before publishing.

---

## Edge Cases

- **Minimal input (just a name and description)**: Generate a full README with placeholder sections marked `<!-- TODO: add example here -->` — better to have a complete skeleton than a short accurate stub
- **Large project (many modules)**: Project Structure shows top-level layout only; note that sub-module docs are in individual module READMEs
- **Multiple service types**: Include all relevant supplements; order them by relevance (most prominent type first)
- **Existing README provided**: Preserve sections that are already complete; improve or add missing sections; flag anything that looks outdated
- **Sensitive values in provided config**: Never include real credentials in the README; replace with clearly labeled placeholders
- **No tests exist**: Note it in Development — do not pretend tests exist
- **CLI with many subcommands**: Show the top 3–5 most-used commands in detail; add "run `--help` for full reference"
- **Agent with many tools**: Show 2–3 representative example interactions; link to a separate `TOOLS.md` if the tool list is large

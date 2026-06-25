---
name: secure-code-generator
description: >
  Use this skill whenever a user asks you to write, generate, implement, create, or scaffold code.
  Triggers include: "write code for", "generate a function", "implement X", "create a module",
  "build a script", "code this up", "write a class for", "create a service that", "scaffold X",
  "add an endpoint for", "write a handler for", "give me a Python script that", or when a user
  describes functionality and expects working code in return. Primary language: Python.
  Always enforces security rules (no hardcoded secrets, env-var config), modularity (single
  responsibility, DRY), and team coding standards before producing any output.
  Apply this skill even for "quick" or "just a snippet" requests — standards apply regardless
  of size. Do NOT apply this skill for code review requests (use code-review-checklist) or
  PR descriptions (use pr-description-generator).
---

# Secure Code Generator Skill

Generates **production-ready Python code** that is secure, modular, and consistent with team
standards. No hardcoded credentials, no configuration buried in source, no silent failures.

---

## Step 1 — Analyze the Request

Before writing any code, identify:

| What to determine | Why it matters |
|---|---|
| What the code must do (inputs, outputs, side effects) | Defines scope and function signatures |
| External dependencies (DB, API, message broker, filesystem) | Every external call needs config + error handling |
| Whether secrets or config values are involved | Drives env-var design |
| Whether this is a standalone script, module, or service component | Drives file/module structure |

If the request is too vague to produce correct code, ask **one focused question** before proceeding.
Do not ask for information that can be reasonably inferred — make sensible defaults and document them.

---

## Step 2 — Security Enforcement (Non-Negotiable)

Read `references/security-rules.md` for full detail. These rules apply to every line of code generated:

### Secrets & Config — Always use environment variables

**Never generate this:**
```python
API_KEY = "sk-abc123"
DB_URL = "postgresql://admin:password@prod-db:5432/mydb"
BASE_URL = "https://internal.example.com"
```

**Always generate this:**
```python
import os

API_KEY = os.environ["API_KEY"]              # fails fast if missing — correct for required secrets
DB_URL = os.environ["DATABASE_URL"]
BASE_URL = os.environ.get("BASE_URL", "https://api.example.com")  # with safe default for non-secret config
```

**For modules with multiple config values, use pydantic-settings:**
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    api_key: str
    database_url: str
    base_url: str = "https://api.example.com"
    timeout_seconds: int = 30

    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"

settings = Settings()
```

### What counts as a secret or config — never hardcode these:
- API keys, tokens, passwords, client secrets
- Database hostnames, ports, names, connection strings
- Internal service URLs, hostnames, IP addresses
- S3 bucket names, queue names, topic names
- Webhook URLs, callback URLs
- Any value that differs between dev/staging/prod

### Input Validation — validate at system boundaries
- Validate all user-supplied input before processing
- Use pydantic models for structured input (HTTP requests, file parsing, CLI args)
- Sanitize file paths to prevent traversal: `pathlib.Path(user_input).resolve()`
- Never pass user input to `eval()`, `exec()`, or `subprocess` with `shell=True`
- Use parameterized queries — never f-strings in SQL

### Logging Safety
- Never log: passwords, tokens, API keys, PII, full payloads that may contain secrets
- Log the event and context, not the secret value:
  ```python
  # Wrong
  logger.info(f"Calling API with key {api_key}")
  # Correct
  logger.info("Calling external API", extra={"endpoint": url, "timeout": timeout})
  ```

---

## Step 3 — Structure & Modularity Rules

### Function design
- One function = one responsibility. If a function does two things, split it.
- Functions should be short enough to read without scrolling (~30 lines max as a guide)
- Name functions as verb phrases: `fetch_deployments()`, `parse_event_payload()`, `send_alert()`
- Accept dependencies (clients, settings) as parameters — makes testing possible without mocking globals

**Prefer:**
```python
def fetch_deployments(client: httpx.Client, base_url: str, cursor: str | None = None) -> list[dict]:
    ...
```

**Over:**
```python
def fetch_deployments():
    client = httpx.Client()   # hidden dependency, untestable
    base_url = "https://hardcoded.example.com"
    ...
```

### Module structure for non-trivial code
```
module_name/
├── __init__.py
├── config.py       # Settings / pydantic-settings only
├── models.py       # Pydantic data models
├── client.py       # External API / DB client wrappers
├── service.py      # Business logic
└── main.py         # Entry point / wiring
```

For simple scripts: a single file is fine — keep config at the top, logic in functions, `if __name__ == "__main__":` at the bottom.

### DRY — Don't Repeat Yourself
- If the same logic appears twice, extract a function
- If the same config value is used in multiple places, define it once in `config.py`
- If the same HTTP client setup appears in multiple modules, create a shared `client.py`

---

## Step 4 — Python Coding Standards

Read `references/python-patterns.md` for patterns and examples.

### Type hints — required on all public functions
```python
def process_events(events: list[dict], max_retries: int = 3) -> tuple[int, list[str]]:
    ...
```

### Error handling — specific exceptions, never silent
```python
# Wrong — swallows everything
try:
    result = call_api()
except Exception:
    pass

# Wrong — bare except
try:
    result = call_api()
except:
    pass

# Correct — specific, logged, re-raised or handled with intent
try:
    result = call_api()
except httpx.TimeoutException as e:
    logger.warning("API call timed out", extra={"url": url, "error": str(e)})
    raise
except httpx.HTTPStatusError as e:
    logger.error("API returned error", extra={"status": e.response.status_code})
    raise
```

### Logging — use the standard library, structured
```python
import logging

logger = logging.getLogger(__name__)   # module-scoped logger, not root logger

logger.info("Processing batch", extra={"batch_size": len(items), "source": source})
logger.warning("Retry attempt", extra={"attempt": attempt, "max": max_retries})
logger.error("Unrecoverable failure", extra={"error": str(e)}, exc_info=True)
```

### Constants — at module level, not inline magic values
```python
# Wrong
if retry_count > 3:
    ...

# Correct
MAX_RETRIES = 3
if retry_count > MAX_RETRIES:
    ...
```

### Imports — organized, no unused
```
# Order: stdlib → third-party → local
import os
import logging
from pathlib import Path

import httpx
from pydantic import BaseModel

from .config import settings
from .models import DeploymentEvent
```

---

## Step 5 — Output Format

Always produce output in this structure:

### 1. Brief summary
One sentence: what the code does and what it depends on.

### 2. Environment Variables Required
List every env var the code uses, with a description and whether it's required or optional:

```
Environment variables:
  API_KEY          (required)  — API authentication token
  DATABASE_URL     (required)  — PostgreSQL connection string
  BASE_URL         (optional, default: https://api.example.com) — API base URL
  TIMEOUT_SECONDS  (optional, default: 30) — HTTP request timeout
```

### 3. The code
Full, working code in a fenced code block. No skeleton, no pseudocode, no `# TODO: implement this`.

### 4. Decisions & Defaults
Note any assumptions made (library choices, default values, retry strategy, etc.) so the user can redirect if needed.

---

## Step 6 — Self-Review Gate

Before finalizing output, mentally scan the generated code against this checklist.
Do not output the checklist — fix any failures silently before responding.

- [ ] No hardcoded credentials, tokens, or API keys anywhere in the code
- [ ] No hardcoded hostnames, service URLs, or database connection strings
- [ ] All external config loaded via `os.environ` or `pydantic-settings`
- [ ] Type hints on all public functions
- [ ] No bare `except:` or `except Exception: pass`
- [ ] All external calls (HTTP, DB, filesystem) have error handling
- [ ] Logging does not contain sensitive values
- [ ] No `eval()`, `exec()`, or `subprocess` with `shell=True` on user input
- [ ] Functions have a single clear responsibility
- [ ] No magic numbers or strings inline — constants defined at module level

If any item fails, fix the code before outputting it.

---

## Edge Cases

- **"Just a quick script"**: Apply all rules. A quick script that hardcodes a key becomes a committed secret. No exceptions.
- **"For local dev only"**: Still use env vars — reading from `.env` via `python-dotenv` or `pydantic-settings` is trivially easy for local use.
- **"I'll add the config later"**: Generate the env-var pattern now. Leave the implementation correct from the start.
- **Large module (>3 files)**: Propose the module structure first and confirm before writing all files.
- **Database access**: Always use parameterized queries. Always close connections (use context managers or connection pools).
- **Async code**: Use `async`/`await` consistently — don't mix sync and async IO calls. Use `asyncio.run()` at the entry point.
- **CLI scripts**: Use `argparse` or `typer`. Config from env vars, flags for runtime overrides.

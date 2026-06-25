# Security Rules for Code Generation

Mandatory rules applied to every piece of generated code. No exceptions for "quick scripts",
"dev-only code", or "I'll fix it later" contexts.

---

## 1. Secrets & Configuration Management

### The Rule
Every value that could differ between environments or that grants access to a resource **must**
come from an environment variable. Never embed it in source.

### What Must Come from Environment Variables

| Category | Examples |
|---|---|
| Authentication | API keys, tokens, client secrets, passwords, bearer tokens |
| Database | Connection strings, hostnames, port numbers, database names, usernames |
| Service endpoints | Internal API URLs, webhook URLs, callback URLs, base URLs |
| Infrastructure | S3 bucket names, SQS queue URLs, Kafka topic names, Redis hosts |
| Feature config | Any value that differs between dev / staging / prod |

### Patterns — Ordered by Preference

**Option 1: pydantic-settings (preferred for modules and services)**
```python
from pydantic_settings import BaseSettings
from pydantic import SecretStr

class Settings(BaseSettings):
    # Required — startup fails with a clear error if missing
    database_url: str
    api_key: SecretStr                   # SecretStr prevents accidental logging

    # Optional with safe defaults
    base_url: str = "https://api.example.com"
    max_retries: int = 3
    timeout_seconds: int = 30
    log_level: str = "INFO"

    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"

settings = Settings()
```

Usage: `settings.api_key.get_secret_value()` — the `.get_secret_value()` call makes it
explicit and intentional every time the raw secret is accessed.

**Option 2: os.environ (acceptable for simple scripts)**
```python
import os

# Required config — raises KeyError with a clear message if not set
DATABASE_URL = os.environ["DATABASE_URL"]
API_KEY = os.environ["API_KEY"]

# Optional config — provide safe default
BASE_URL = os.environ.get("BASE_URL", "https://api.example.com")
TIMEOUT = int(os.environ.get("TIMEOUT_SECONDS", "30"))
```

**Never use:**
```python
# Direct string embedding
API_KEY = "sk-abc123"
DB_PASSWORD = "mysecretpassword"
BASE_URL = "https://internal-service.prod.example.com"

# os.environ with a secret default
API_KEY = os.environ.get("API_KEY", "sk-fallback-key")   # fallback is itself a hardcoded secret
```

### .env Files (Local Development)
Generated `.env.example` files should contain placeholder keys, never real values:
```bash
# .env.example — commit this
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
API_KEY=your-api-key-here
BASE_URL=https://api.example.com

# .env — never commit this (add to .gitignore)
```

Always include `.env` in `.gitignore`. If generating a script that uses python-dotenv:
```python
from dotenv import load_dotenv
load_dotenv()   # loads .env if present; env vars already set take precedence
```

---

## 2. Input Validation & Injection Prevention

### SQL — Always Use Parameterized Queries

```python
# Wrong — SQL injection vulnerability
cursor.execute(f"SELECT * FROM users WHERE email = '{user_email}'")

# Correct — parameterized
cursor.execute("SELECT * FROM users WHERE email = %s", (user_email,))

# With SQLAlchemy ORM (preferred)
stmt = select(User).where(User.email == user_email)
result = session.execute(stmt)
```

### File Paths — Prevent Path Traversal

```python
from pathlib import Path

BASE_DIR = Path(os.environ["DATA_DIR"]).resolve()

def safe_read_file(filename: str) -> str:
    target = (BASE_DIR / filename).resolve()
    if not str(target).startswith(str(BASE_DIR)):
        raise ValueError(f"Path traversal attempt blocked: {filename}")
    return target.read_text()
```

### Subprocess — Never shell=True with User Input

```python
import subprocess

# Wrong — command injection if user_input is malicious
subprocess.run(f"ls {user_input}", shell=True)

# Correct — argument list, shell=False (default)
subprocess.run(["ls", user_input], check=True, capture_output=True)
```

### Deserialization — Safe Loaders Only

```python
import yaml
import json

# Wrong — arbitrary code execution via PyYAML
data = yaml.load(user_content)

# Correct
data = yaml.safe_load(user_content)    # YAML
data = json.loads(user_content)         # JSON — always safe

# Never use pickle on untrusted data
# import pickle; pickle.loads(user_data)  — remote code execution risk
```

### Pydantic for Structured Input Validation

```python
from pydantic import BaseModel, Field, validator

class CreateDeploymentRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=64, pattern=r"^[a-z][a-z0-9-]*$")
    namespace: str = Field(..., min_length=1, max_length=63)
    replicas: int = Field(default=1, ge=1, le=100)
    image: str = Field(..., min_length=1, max_length=256)
```

---

## 3. Logging Safety

### What to Never Log

- Passwords, tokens, API keys, client secrets
- Full database connection strings (contain credentials)
- Personal identifiable information (PII): emails, names, SSNs, phone numbers
- Full HTTP request/response bodies when they may contain credentials
- Stack traces that expose internal system details to end users (log them server-side, return generic error to caller)

### Safe Logging Pattern

```python
import logging

logger = logging.getLogger(__name__)

# Wrong — logs the secret
logger.info(f"Authenticating with key: {api_key}")

# Correct — log the action, not the secret
logger.info("Authenticating with external API", extra={"service": "payments", "user_id": user_id})

# For HTTP clients — log method/URL/status, not auth headers or body
logger.info(
    "HTTP request completed",
    extra={"method": "POST", "url": url, "status": response.status_code, "duration_ms": elapsed}
)
```

### Redacting Sensitive Fields

When you must log a dict that may contain secrets, redact before logging:
```python
SENSITIVE_KEYS = {"password", "token", "api_key", "secret", "authorization"}

def redact(data: dict) -> dict:
    return {k: "***REDACTED***" if k.lower() in SENSITIVE_KEYS else v for k, v in data.items()}

logger.debug("Request payload", extra={"payload": redact(request_data)})
```

---

## 4. Dependencies & Third-Party Libraries

- Always pin versions in `requirements.txt` or `pyproject.toml`:
  ```
  httpx==0.27.0
  pydantic==2.7.1
  pydantic-settings==2.3.0
  ```
- Prefer well-maintained, widely-used libraries over obscure alternatives
- Do not add a dependency for something the stdlib handles adequately
- When using `requests` in new code, prefer `httpx` (supports async, better timeouts)

---

## 5. HTTP Client Security

```python
import httpx

# Always set explicit timeouts — never rely on defaults (can hang indefinitely)
client = httpx.Client(
    base_url=settings.base_url,
    timeout=httpx.Timeout(connect=5.0, read=30.0, write=10.0, pool=5.0),
    headers={"Authorization": f"Bearer {settings.api_key.get_secret_value()}"},
)

# Verify SSL by default (verify=True is default in httpx — never set verify=False in production)
# If you must disable for dev, guard it:
verify_ssl = os.environ.get("DISABLE_SSL_VERIFY", "false").lower() != "true"
client = httpx.Client(verify=verify_ssl)
```

# Python Patterns Reference

Reusable patterns for generating consistent, modular, testable Python code.

---

## 1. Module Layout

### Simple Script (single file)
```
task_name.py
.env.example
requirements.txt
```

Structure inside the file:
```python
"""One-line description of what this script does."""

# 1. Stdlib imports
import logging
import os
from pathlib import Path

# 2. Third-party imports
import httpx
from pydantic_settings import BaseSettings

# 3. Config
class Settings(BaseSettings):
    ...

settings = Settings()

# 4. Logger
logger = logging.getLogger(__name__)

# 5. Constants
MAX_RETRIES = 3
BATCH_SIZE = 100

# 6. Models / dataclasses

# 7. Functions (helpers first, main logic last)

# 8. Entry point
if __name__ == "__main__":
    logging.basicConfig(level=settings.log_level)
    main()
```

### Service / Module (multi-file)
```
myservice/
├── __init__.py
├── config.py       # BaseSettings — imported by all other modules
├── models.py       # Pydantic request/response/domain models
├── client.py       # HTTP/DB client setup and wrappers
├── repository.py   # Data access layer (DB queries)
├── service.py      # Business logic — no I/O, calls repository/client
├── api.py          # FastAPI routes / handler wiring
└── main.py         # App factory and startup

tests/
├── conftest.py
├── test_service.py
└── test_api.py
```

---

## 2. Config Pattern (pydantic-settings)

```python
# config.py
from pydantic_settings import BaseSettings
from pydantic import SecretStr
from functools import lru_cache

class Settings(BaseSettings):
    # Database
    database_url: str

    # External APIs
    api_key: SecretStr
    api_base_url: str = "https://api.example.com"
    api_timeout_seconds: int = 30

    # App behavior
    log_level: str = "INFO"
    environment: str = "development"
    max_retries: int = 3

    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"
        case_sensitive = False   # API_KEY and api_key both work

@lru_cache(maxsize=1)
def get_settings() -> Settings:
    return Settings()
```

Usage across modules:
```python
from .config import get_settings

settings = get_settings()
```

---

## 3. HTTP Client Pattern (httpx)

```python
# client.py
from contextlib import contextmanager
from typing import Generator
import httpx
from .config import get_settings

settings = get_settings()

def build_client() -> httpx.Client:
    return httpx.Client(
        base_url=settings.api_base_url,
        timeout=httpx.Timeout(
            connect=5.0,
            read=settings.api_timeout_seconds,
            write=10.0,
            pool=5.0,
        ),
        headers={
            "Authorization": f"Bearer {settings.api_key.get_secret_value()}",
            "Content-Type": "application/json",
        },
    )

@contextmanager
def get_client() -> Generator[httpx.Client, None, None]:
    with build_client() as client:
        yield client
```

Usage:
```python
from .client import get_client

with get_client() as client:
    response = client.get("/v1/resources")
    response.raise_for_status()
```

### Async variant (httpx.AsyncClient)

```python
import httpx
from contextlib import asynccontextmanager

@asynccontextmanager
async def get_async_client():
    async with httpx.AsyncClient(
        base_url=settings.api_base_url,
        timeout=httpx.Timeout(read=30.0),
        headers={"Authorization": f"Bearer {settings.api_key.get_secret_value()}"},
    ) as client:
        yield client
```

---

## 4. Error Handling Pattern

### Layer responsibility
- **client.py / repository.py**: Catch low-level errors, log them, re-raise as domain exceptions
- **service.py**: Catch domain exceptions, apply business rules (retry, fallback)
- **api.py / main.py**: Catch domain exceptions, translate to HTTP responses or exit codes

```python
# Domain exceptions — define in models.py or exceptions.py
class ServiceUnavailableError(Exception):
    """Raised when a downstream service is unreachable after retries."""

class ResourceNotFoundError(Exception):
    """Raised when a requested resource does not exist."""

# client.py — translate transport errors to domain errors
def get_deployment(client: httpx.Client, name: str) -> dict:
    try:
        response = client.get(f"/v1/deployments/{name}")
        if response.status_code == 404:
            raise ResourceNotFoundError(f"Deployment '{name}' not found")
        response.raise_for_status()
        return response.json()
    except httpx.TimeoutException as e:
        logger.warning("Timeout fetching deployment", extra={"name": name})
        raise ServiceUnavailableError(f"Timed out fetching deployment '{name}'") from e
    except httpx.HTTPStatusError as e:
        logger.error("HTTP error", extra={"status": e.response.status_code, "name": name})
        raise
```

### Retry pattern (with exponential backoff)
```python
import time
import logging

logger = logging.getLogger(__name__)

def with_retry(fn, max_retries: int = 3, base_delay: float = 1.0):
    for attempt in range(1, max_retries + 1):
        try:
            return fn()
        except ServiceUnavailableError as e:
            if attempt == max_retries:
                raise
            delay = base_delay * (2 ** (attempt - 1))
            logger.warning(
                "Retrying after failure",
                extra={"attempt": attempt, "max": max_retries, "delay_s": delay},
            )
            time.sleep(delay)
```

---

## 5. Database Pattern (SQLAlchemy)

```python
# repository.py
from contextlib import contextmanager
from typing import Generator
from sqlalchemy import create_engine, text
from sqlalchemy.orm import Session, sessionmaker
from .config import get_settings

settings = get_settings()

engine = create_engine(
    settings.database_url,
    pool_pre_ping=True,   # detect stale connections
    pool_size=5,
    max_overflow=10,
)
SessionLocal = sessionmaker(bind=engine, autocommit=False, autoflush=False)

@contextmanager
def get_session() -> Generator[Session, None, None]:
    session = SessionLocal()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()
```

Usage:
```python
from .repository import get_session
from .models import Deployment

def list_deployments(namespace: str) -> list[Deployment]:
    with get_session() as session:
        return session.query(Deployment).filter_by(namespace=namespace).all()
```

---

## 6. Type Hints Reference

```python
from typing import Optional
from collections.abc import Generator, AsyncGenerator

# Basic
def process(name: str, count: int, active: bool) -> dict[str, int]:
    ...

# Optional values (Python 3.10+ prefer X | None over Optional[X])
def find_user(user_id: str) -> dict | None:
    ...

# Lists and dicts
def batch_process(items: list[str]) -> list[dict]:
    ...

def merge_configs(base: dict[str, str], overrides: dict[str, str]) -> dict[str, str]:
    ...

# Callables
from collections.abc import Callable
def apply(fn: Callable[[str], bool], items: list[str]) -> list[str]:
    ...

# Generators
def stream_records(cursor: str | None = None) -> Generator[dict, None, None]:
    ...
```

---

## 7. FastAPI Endpoint Pattern

```python
# api.py
from fastapi import APIRouter, Depends, HTTPException, Query
from .models import DeploymentResponse, CreateDeploymentRequest, PaginatedResponse
from .service import DeploymentService
from .exceptions import ResourceNotFoundError

router = APIRouter(prefix="/v1/deployments", tags=["deployments"])

@router.get("/", response_model=PaginatedResponse[DeploymentResponse])
def list_deployments(
    namespace: str = Query(..., description="Kubernetes namespace"),
    cursor: str | None = Query(None, description="Pagination cursor"),
    limit: int = Query(20, ge=1, le=100),
    service: DeploymentService = Depends(),
):
    return service.list(namespace=namespace, cursor=cursor, limit=limit)

@router.get("/{name}", response_model=DeploymentResponse)
def get_deployment(name: str, namespace: str = Query(...), service: DeploymentService = Depends()):
    try:
        return service.get(name=name, namespace=namespace)
    except ResourceNotFoundError:
        raise HTTPException(status_code=404, detail=f"Deployment '{name}' not found in '{namespace}'")
```

---

## 8. CLI Script Pattern (argparse)

```python
import argparse
import logging
import os

def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(description="Process deployment events")
    parser.add_argument("--namespace", required=True, help="Kubernetes namespace to process")
    parser.add_argument("--dry-run", action="store_true", help="Print actions without executing")
    parser.add_argument("--limit", type=int, default=100, help="Max records to process")
    return parser.parse_args()

def main() -> None:
    args = parse_args()
    logging.basicConfig(
        level=os.environ.get("LOG_LEVEL", "INFO"),
        format="%(asctime)s %(levelname)s %(name)s %(message)s",
    )
    logger = logging.getLogger(__name__)
    logger.info("Starting", extra={"namespace": args.namespace, "dry_run": args.dry_run})
    # ... implementation

if __name__ == "__main__":
    main()
```

---

## 9. Async Entry Point Pattern

```python
import asyncio
import logging

async def main() -> None:
    logging.basicConfig(level="INFO")
    logger = logging.getLogger(__name__)

    async with get_async_client() as client:
        results = await process_all(client)
        logger.info("Done", extra={"count": len(results)})

if __name__ == "__main__":
    asyncio.run(main())
```

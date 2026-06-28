# Base README Template

Generic template that applies to every service type.
Type-specific supplement sections are in `supplements.md`.
Replace all `<!-- placeholder -->` comments with real content.

---

# <Service Name>

> <One sentence: what it does and the primary outcome it delivers.>

<!-- Optional badges — only include if repo/CI info is available
![Python](https://img.shields.io/badge/python-3.11+-blue)
![License](https://img.shields.io/badge/license-MIT-green)
-->

---

## Overview

<2–4 sentences describing what this service does, what problem it solves, and who uses it.
For agents: mention the LLM provider and what the agent is specialized for.
For networking apps: mention the Cisco platforms and what operations it supports.>

---

## Quick Start

### Prerequisites

- Python 3.11+
- `<any other required tools: e.g., access to a vManage instance, Anthropic API key, etc.>`

### Install

```bash
# Clone the repository
git clone <repo-url>
cd <repo-name>

# Create a virtual environment
python -m venv .venv
source .venv/bin/activate       # Linux / macOS
# .venv\Scripts\activate        # Windows

# Install dependencies
pip install -r requirements.txt
```

### Configure

```bash
# Copy the example env file
cp .env.example .env

# Edit .env and fill in required values
# See Configuration section for details
```

### Run

```bash
<command to start the service>
```

### Verify

```bash
<command to confirm the service is running correctly>
# Expected output:
# <what successful output looks like>
```

---

## Configuration

All configuration is loaded from environment variables. Copy `.env.example` to `.env` and fill in required values.

### Environment Variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `<VAR_NAME>` | Yes | — | <what it controls and where to get it> |
| `<VAR_NAME>` | Yes | — | <description> |
| `<VAR_NAME>` | No | `<default>` | <description> |

### `.env.example`

```bash
# --- Required ---
<VAR_NAME>=<your-value-here>
<VAR_NAME>=<your-value-here>

# --- Optional ---
<VAR_NAME>=<default-value>
<VAR_NAME>=<default-value>
```

---

## Usage

### <Most Common Use Case>

```bash
<command>
```

Expected output:
```
<what success looks like>
```

### <Second Common Use Case>

```bash
<command>
```

---

## Project Structure

```
<service-name>/
├── <file>.py          # <one-line purpose>
├── <file>.py          # <one-line purpose>
├── <directory>/       # <one-line purpose>
│   └── ...
├── tests/
│   └── ...
├── .env.example       # template for environment variables
└── requirements.txt
```

---

## Development

### Local Setup

```bash
# Install dev dependencies
pip install -r requirements-dev.txt   # or: pip install -e ".[dev]"

# Copy env file
cp .env.example .env
# Fill in .env with development values
```

### Run Tests

```bash
pytest tests/ -v
```

Expected: all tests pass.

### Linting & Formatting

```bash
# Format
ruff format .

# Lint
ruff check .

# Type check
mypy .
```

### Contributing

1. Create a branch: `git checkout -b <your-branch>`
2. Make changes and add tests
3. Run the test suite and linting before committing
4. Open a pull request — see PR description guidelines in `.github/` if present

---

## License

<!-- Add license info here, or remove this section if not applicable -->

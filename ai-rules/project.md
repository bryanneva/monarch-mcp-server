---
description: Project instructions
alwaysApply: true
---

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Security Warning: Financial Data

**This is a financial application. Sensitive data must NEVER be sent through Anthropic servers.**

- **NEVER** ask users for their Monarch Money credentials (email, password, MFA codes) through Claude
- **NEVER** log, display, or process authentication tokens in tool outputs
- Credentials are entered via `login_setup.py` in the user's terminal, stored in the system keyring, and never exposed to Claude
- If a user shares financial data (account numbers, balances, transaction details), remind them this data is transmitted to Anthropic's servers

## Build and Development Commands

```bash
# Install dependencies
pip install -r requirements.txt
pip install -e .

# Install with dev dependencies
pip install -e ".[dev]"

# Run tests
pytest

# Run single test file
pytest tests/test_validation.py

# Run single test
pytest tests/test_validation.py::test_validate_limit_valid

# Type checking
mypy src/

# Format code
black src/ tests/
isort src/ tests/

# Run the MCP server directly (for testing)
python src/monarch_mcp_server/server.py
```

## Architecture

This is a Model Context Protocol (MCP) server that connects Claude Desktop to the Monarch Money personal finance API.

```
Claude Desktop → MCP Server (server.py) → MonarchMoney API (api.monarch.com)
                       ↓
              System Keyring (secure_session.py)
```

### Key Components

- **`src/monarch_mcp_server/server.py`**: Main MCP server using FastMCP. Defines all tools (`get_accounts`, `get_transactions`, etc.). Contains input validation, error sanitization, and the async wrapper (`run_async`) that executes MonarchMoney API calls with timeouts.

- **`src/monarch_mcp_server/secure_session.py`**: Manages authentication tokens via the system keyring (`keyring` library). Never stores credentials in files.

- **`login_setup.py`**: Standalone CLI script for one-time authentication. Users run this in their terminal (not through Claude) to authenticate with MFA support. Saves session token to keyring.

### Authentication Flow

1. User runs `python login_setup.py` in terminal
2. Enters credentials via `getpass` (hidden input)
3. Handles MFA if enabled
4. Token saved to system keyring (`com.mcp.monarch-mcp-server`)
5. MCP server loads token from keyring on each request
6. Fallback: environment variables `MONARCH_EMAIL`/`MONARCH_PASSWORD` (less secure, logs warning)

### API Domain Fix

The MonarchMoney library's default endpoint was changed. This server patches it:
```python
MonarchMoneyEndpoints.BASE_URL = "https://api.monarch.com"
```
This fix appears in both `server.py` and `login_setup.py`.

### Error Handling

All tool functions use `_sanitize_error()` to prevent internal details (API URLs, tokens, stack traces) from leaking to users. Only user-actionable messages are returned.

### Input Validation

- `limit`: 1-1000
- `offset`: non-negative
- `date`: YYYY-MM-DD format (regex validated)
- `description`: max 500 characters

## Testing

Tests are in `tests/` and use pytest with pytest-asyncio. Key test files:
- `test_validation.py`: Input validation edge cases
- `test_error_handling.py`: Error sanitization
- `test_tools.py`: Tool function behavior
- `test_run_async.py`: Async wrapper and timeout handling

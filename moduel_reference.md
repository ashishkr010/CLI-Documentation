# CLI Tool - Module Reference & Development Guide

## Core Modules Overview

This document provides detailed reference for all major modules in the CLI Tool codebase.

---

## Module Dependency Graph

```
┌─────────────────────────────────────────────┐
│              api.py                         │
│       (FastAPI Application Entry)           │
└──────────────┬──────────────────────────────┘
               │
               ├──→ routes.py (API Endpoints)
               │    │
               │    ├──→ executor.py (Command Execution)
               │    │    ├──→ registry.py (Command Metadata)
               │    │    ├──→ metadata.py (Metadata Generation)
               │    │    ├──→ decision.py (Analysis)
               │    │    ├──→ hints.py (AI Hints)
               │    │    ├──→ remote.py (SSH Execution)
               │    │    └──→ ssh.py (SSH Client)
               │    │
               │    ├──→ session.py (Session Management)
               │    │
               │    └──→ tool_result_adapter.py (Result Formatting)
               │
               ├──→ models.py (Pydantic Models)
               │
               ├──→ auth.py (Authentication)
               │
               └──→ config/settings.py (Configuration)


Local CLI Interfaces:
├──→ cli.py (Main CLI Group)
├──→ meta_cli.py (Metadata CLI)
├──→ batch.py (Batch Processing)
├──→ chain.py (Command Chaining)
└──→ repl.py (Interactive Shell)


Support Modules:
├──→ logger.py (Logging Configuration)
├──→ audit.py (Audit Logging)
├──→ telemetry.py (Telemetry/Monitoring)
├──→ pool.py (Connection Pool)
├──→ utils.py (Utilities)
└──→ remote.py (Remote Execution)
```

---

## executor.py - Core Command Execution

**File:** `/local_cmd_executor/executor.py`

**Responsibility:** Primary interface for executing commands with full metadata support.

### Classes

#### CommandExecutor
Main executor class for local command execution.

**Key Methods:**

| Method | Purpose | Parameters | Returns |
|--------|---------|-----------|---------|
| `__init__` | Initialize executor | `enable_metadata=True` | - |
| `execute_command` | Simple local execution | `command, cwd, timeout` | `(success, stdout, stderr, exit_code)` |
| `execute` | Execute with metadata | `command, metadata_format` | `dict with result and metadata` |
| `detect_command_type` | Analyze command structure | `command` | `dict with type, needs_shell, etc` |
| `_execute_with_retry` | Resilient execution | `command, timeout` | `dict with result` |

**Usage Example:**
```python
from local_cmd_executor.executor import CommandExecutor

executor = CommandExecutor(enable_metadata=True)

# Simple execution
result = executor.execute_command("ls -la /tmp")
success, stdout, stderr, exit_code = result

# With metadata
result = executor.execute("ls -la /tmp", metadata_format="summary")
print(result['stdout'])
print(result['metadata'])
```

#### UnifiedExecutor
Combined interface for both local and remote execution.

**Key Methods:**

| Method | Purpose |
|--------|---------|
| `execute` | Route to local or remote execution |
| `_execute_local` | Local execution path |
| `_execute_remote` | Remote execution path |

---

## remote.py - SSH Remote Execution

**File:** `/local_cmd_executor/remote.py`

**Responsibility:** SSH-based remote command execution with persistent connections.

### Classes

#### RemoteExecutor
Context manager for SSH connections with batch execution support.

**Key Methods:**

| Method | Purpose | Parameters | Returns |
|--------|---------|-----------|---------|
| `__init__` | Initialize SSH connection | `host, username, password, key_file, timeout` | - |
| `connect` | Establish SSH connection | - | None |
| `close` | Close SSH connection | - | None |
| `execute_command` | Execute single command | `command, timeout` | `(stdout, stderr, exit_code)` |
| `execute_commands` | Execute multiple commands | `commands: List[str]` | `List[tuple]` |
| `is_connected` | Check connection status | - | `bool` |

**Usage Example:**
```python
from local_cmd_executor.remote import RemoteExecutor

# Context manager usage
with RemoteExecutor(
    host="server.example.com",
    username="deploy",
    password="secret",
    timeout=30
) as executor:
    stdout, stderr, code = executor.execute_command("pwd")
    print(f"Current directory: {stdout.strip()}")
```

---

## routes.py - API Endpoints

**File:** `/routes.py`

**Responsibility:** FastAPI route definitions and HTTP endpoint implementations.

### Route Groups

#### Base Router (`/api`)
Core command execution endpoints.

**Endpoints:**
- `POST /run-command` - Execute local command
- `POST /local-command-executor` - Execute remote command
- `GET /ssh-sessions` - List SSH sessions
- `DELETE /ssh-sessions/{session_key}` - Close SSH session
- `POST /process-manager` - Manage background processes

#### Session Router (`/api/session`)
Session lifecycle management.

**Endpoints:**
- `POST /create-session` - Create new session
- `GET /session/{session_id}` - Get session info
- `DELETE /session/{session_id}` - Close session
- `GET /session/{session_id}/history` - Get command history
- `GET /session/{session_id}/suggestions` - Get AI suggestions

#### Metadata Router (`/api`)
Advanced result processing.

**Endpoints:**
- `POST /tool-result` - Process tool result with metadata

#### Registry Router (`/api`)
Command registry management.

**Endpoints:**
- `GET /registry` - Get registered commands

---

## session.py - Session Management

**File:** `/local_cmd_executor/session.py`

**Responsibility:** Manage command execution sessions with state and history.

### Classes

#### Session
Single execution session with history and context.

**Key Attributes:**
- `id: str` - Unique session identifier
- `created_at: float` - Creation timestamp
- `last_used_at: float` - Last activity timestamp
- `timeout: int` - Session timeout in seconds
- `command_history: List[Dict]` - Recorded commands
- `current_directory: str` - Working directory
- `context: Optional[Session]` - Context reference

**Key Methods:**

| Method | Purpose | Returns |
|--------|---------|---------|
| `record_command` | Log command execution | None |
| `is_expired` | Check if session timed out | `bool` |
| `touch` | Update last activity | None |
| `_save_history` | Persist history to disk | None |
| `_load_history` | Load history from disk | None |

**Usage Example:**
```python
from local_cmd_executor.session import SessionManager

manager = SessionManager(
    session_timeout=3600,
    cleanup_interval=300,
    context_enabled=True
)

# Create session
session = manager.create_session()
print(f"Session ID: {session.id}")

# Record command
session.record_command(
    command="ls",
    metadata={...},
    result={"exit_code": 0, "success": True}
)

# Query history
history = session.command_history
```

#### SessionManager
Manages multiple sessions with cleanup.

**Key Methods:**

| Method | Purpose |
|--------|---------|
| `create_session` | Create new session |
| `get_session` | Retrieve session by ID |
| `close_session` | Close and cleanup session |
| `list_sessions` | Get all active sessions |
| `cleanup_expired` | Remove timed-out sessions |

---

## registry.py - Command Registry

**File:** `/local_cmd_executor/registry.py`

**Responsibility:** Central repository of command metadata and intelligence.

### Classes

#### CommandRegistry
Hierarchical registry of command metadata.

**Key Methods:**

| Method | Purpose | Parameters | Returns |
|--------|---------|-----------|---------|
| `register_command` | Add command metadata | `command, metadata` | None |
| `get_command_metadata` | Lookup command | `command` | `Optional[Dict]` |
| `get_commands_by_category` | Search by category | `category` | `List[str]` |
| `get_related_commands` | Find related commands | `command` | `List[str]` |

**Metadata Structure:**
```python
{
    "description": "Command description",
    "category": "system.process",
    "output_format": "tabular",  # tabular, json, text, lines
    "risk_level": "low",  # low, medium, high
    "common_options": [
        {"flag": "-l", "description": "Long format"},
        {"flag": "-a", "description": "All files"}
    ],
    "related_commands": ["ls", "find"],
    "examples": [
        {"command": "ls -la", "description": "List all with details"}
    ]
}
```

**Usage Example:**
```python
from local_cmd_executor.registry import CommandRegistry

registry = CommandRegistry()

# Register new command
registry.register_command("custom-tool", {
    "description": "Custom tool",
    "category": "custom.tools",
    "output_format": "json",
    "risk_level": "low"
})

# Lookup
metadata = registry.get_command_metadata("ls")

# Category search
commands = registry.get_commands_by_category("system.process")
```

---

## metadata.py - Metadata Generation

**File:** `/local_cmd_executor/metadata.py`

**Responsibility:** Generate rich metadata for command execution.

### Classes

#### CommandMetadata
Generates comprehensive metadata for commands.

**Key Methods:**

| Method | Purpose |
|--------|---------|
| `update_after_execution` | Enrich with execution results |
| `_analyze_output` | Parse output structure |
| `_suggest_next_steps` | Recommend follow-up commands |
| `_infer_basic_intent` | Determine command purpose |

**Generated Metadata:**
```python
{
    "command": "ls -la /tmp",
    "base_command": "ls",
    "execution_context": {
        "user": "ashish",
        "host": "localhost",
        "platform": "Linux"
    },
    "result": {
        "exit_code": 0,
        "success": True,
        "execution_time_ms": 45
    },
    "registry": {
        "description": "List directory contents",
        "category": "filesystem.content",
        "output_format": "tabular",
        "risk_level": "low"
    },
    "output_analysis": {
        "format_type": "tabular",
        "line_count": 42,
        "columns": ["permissions", "owner", "size", "date", "name"]
    },
    "next_steps": ["grep", "find", "cd"]
}
```

---

## decision.py - Command Analysis

**File:** `/local_cmd_executor/decision.py`

**Responsibility:** Intelligent command analysis and alternative suggestions.

### Classes

#### CommandDecision
Analyzes commands and suggests alternatives.

**Key Methods:**

| Method | Purpose |
|--------|---------|
| `analyze_command` | Perform full analysis |
| `_analyze_decision_factors` | Score command factors |
| `_suggest_alternatives` | Suggest better commands |
| `_assess_risk` | Evaluate risk level |

**Analysis Output:**
```python
{
    "command": "sudo apt-get update",
    "base_command": "sudo",
    "decision_factors": {
        "command_familiarity": 0.95,
        "argument_validity": 0.88,
        "context_appropriateness": 0.85,
        "risk_level": "high",
        "efficiency": 0.7
    },
    "alternatives": [
        {
            "alternative": "apt update",
            "reasoning": "Shorter syntax",
            "confidence": 0.92
        }
    ],
    "risk_assessment": {
        "privilege_level": "elevated",
        "system_impact": "package_management",
        "requires_confirmation": True
    }
}
```

---

## hints.py - AI/LLM Optimization

**File:** `/local_cmd_executor/hints.py`

**Responsibility:** Generate hints for AI systems to better process outputs.

### Classes

#### ModelHints
Provides parsing and optimization hints for AI.

**Key Methods:**

| Method | Purpose |
|--------|---------|
| `generate_hints` | Create comprehensive hints |
| `_determine_parsing_strategy` | Best parsing approach |
| `_determine_output_format` | Output structure |
| `_suggest_token_efficiency` | Reduce token usage |

**Hint Structure:**
```python
{
    "command": "ls -la /tmp",
    "parsing_strategy": {
        "name": "tabular_parse",
        "description": "Parse as table with columns",
        "approach": "delimited_columns",
        "primary_delimiter": " ",
        "skip_lines": 1
    },
    "output_format": "tabular",
    "token_efficiency": {
        "recommendation": "Parse structured output first",
        "typical_overhead": "minimal",
        "approach": "Extract key columns only"
    },
    "confidence": 0.95,
    "specific_hints": [
        {
            "pattern": "permissions",
            "parse_as": "string"
        }
    ]
}
```

---

## logger.py - Logging System

**File:** `/local_cmd_executor/logger.py`

**Responsibility:** Centralized logging configuration.

### Functions

#### get_logger
Get logger for specific mode.

```python
from local_cmd_executor.logger import get_logger

api_logger = get_logger(mode="api")
cmd_logger = get_logger(mode="command")
sys_logger = get_logger(mode="system")

api_logger.info("API started")
cmd_logger.info("Command executed")
sys_logger.error("System error")
```

**Logger Modes:**
- `api` - API request/response logging
- `command` - Command execution logging
- `system` - System-level events
- `metadata` - Metadata generation
- `decision` - Decision analysis

---

## audit.py - Audit Logging

**File:** `/local_cmd_executor/audit.py`

**Responsibility:** Security audit trail logging.

### Functions

#### audit_logger
Centralized audit logging.

```python
from local_cmd_executor.audit import audit_logger

# Log connection
audit_logger.log_connection(
    action="connect",
    host="server.com",
    username="deploy",
    success=True
)

# Log command
audit_logger.log_command(
    command="ls -la",
    user="ashish",
    session_id="abc-123",
    exit_code=0,
    success=True
)

# Log system event
audit_logger.log_system_event(
    event_type="session_created",
    details={"session_id": "abc-123"}
)
```

---

## cli.py - CLI Interface

**File:** `/local_cmd_executor/cli.py`

**Responsibility:** Click-based command-line interface.

### Commands

#### run
Execute a command with optional metadata.

```bash
lcmd run "ls -la /tmp"
lcmd run "pwd" --show-metadata --metadata-format summary
lcmd run "npm start" --output-format json
```

#### batch
Execute commands in batch.

```bash
lcmd batch --file commands.txt
```

#### chain
Chain commands with operators.

```bash
lcmd chain "cd /tmp && ls -la && pwd"
```

#### repl
Interactive shell.

```bash
lcmd repl
```

---

## models.py - Request/Response Models

**File:** `/models.py`

**Responsibility:** Pydantic data models for API validation.

### Models

#### CommandRequest
```python
class CommandRequest(BaseModel):
    command: str  # Required
    remote: bool = False
    host: Optional[str] = None
    port: int = 22
    username: Optional[str] = None
    password: Optional[str] = None
    timeout: Optional[int] = 30
    session_id: Optional[str] = None
```

#### CommandResponse
```python
class CommandResponse(BaseModel):
    success: bool
    exit_code: int
    stdout: str
    stderr: str
    command: str
    execution_time: Optional[float]
    metadata: Optional[Dict[str, Any]]
    model_hints: Optional[Dict[str, Any]]
    decision: Optional[Dict[str, Any]]
```

#### SessionRequest / SessionResponse
```python
class SessionRequest(BaseModel):
    session_id: Optional[str]
    client_info: Optional[Dict[str, Any]]

class SessionResponse(BaseModel):
    session_id: str
    created: bool
```

---

## config/settings.py - Configuration

**File:** `/config/settings.py`

**Responsibility:** Application settings management.

### Settings Class

```python
class Settings:
    API_SERVER_URL: str = "http://localhost:7304"
    FASTAPI_PORT: int = 7304
    # Additional config from environment
```

**Configuration Sources:**
1. Environment variables (highest priority)
2. .env file
3. Default values (lowest priority)

---

## auth.py - Authentication

**File:** `/auth.py`

**Responsibility:** Authentication and authorization.

### Functions

#### auth_required
Dependency for protected endpoints.

```python
@app.post("/protected")
async def protected_endpoint(user = Depends(auth_required())):
    # user contains authenticated user info
    pass
```

**Current Status:** Disabled for development (dummy user returned).

---

## pool.py - Connection Pooling

**File:** `/local_cmd_executor/pool.py`

**Responsibility:** Manage connection pools.

### Classes

#### ConnectionPool
Generic connection pool with lifecycle management.

**Key Methods:**
- `get_connection` - Get or create connection
- `release_connection` - Return connection to pool
- `cleanup` - Remove idle connections

#### SSHConnectionPool (in routes.py)
Specialized pool for SSH connections.

---

## batch.py - Batch Processing

**File:** `/local_cmd_executor/batch.py`

**Responsibility:** Execute multiple commands in sequence.

### Classes

#### BatchProcessor
Process multiple commands with shared context.

**Key Methods:**
- `execute_batch` - Run multiple commands
- `execute_file` - Execute from file

---

## chain.py - Command Chaining

**File:** `/local_cmd_executor/chain.py`

**Responsibility:** Support command chaining with operators.

### Classes

#### CommandChain
Parse and execute chained commands.

---

## repl.py - Interactive Shell

**File:** `/local_cmd_executor/repl.py`

**Responsibility:** Interactive command shell interface.

### Classes

#### CommandREPL
Interactive shell with history and context.

---

## Development Guide

### Adding a New Command Handler

1. **Register in CommandRegistry:**
```python
registry.register_command("new-cmd", {
    "description": "...",
    "category": "...",
    "output_format": "...",
    "risk_level": "..."
})
```

2. **Add Special Handling (if needed):**
```python
def _handle_new_cmd_command(self, command: str, timeout: int) -> dict:
    # Custom implementation
    return result
```

3. **Register Handler:**
```python
if "new-cmd" in command:
    return self._handle_new_cmd_command(command, timeout)
```

### Adding New API Endpoint

1. **Define Request Model:**
```python
class NewRequest(BaseModel):
    param1: str
    param2: int
```

2. **Define Response Model:**
```python
class NewResponse(BaseModel):
    result: str
```

3. **Create Endpoint:**
```python
@router.post("/new-endpoint")
async def new_endpoint(request: NewRequest, user = Depends(auth_required())):
    # Implementation
    return NewResponse(result="...")
```

### Testing

Run tests:
```bash
pytest tests/

# Specific test
pytest tests/test_executor.py::test_simple_command
```

---

## Performance Considerations

### Optimization Tips

1. **Command Execution:**
   - Use subprocess directly for simple commands
   - Cache registry lookups
   - Reuse SSH connections

2. **Metadata Generation:**
   - Make optional (only on request)
   - Cache registry data
   - Limit analysis depth

3. **Session Management:**
   - Implement session timeout
   - Cleanup expired sessions
   - Archive old history

4. **Logging:**
   - Use async logging
   - Implement log rotation
   - Compress old logs


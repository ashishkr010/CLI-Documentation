# CLI Tool Architecture & Implementation Guide

**Version:** 1.0.0  
**Last Updated:** December 2025

## Table of Contents

1. [Technology Stack Overview](#technology-stack-overview)
2. [Architectural Design](#architectural-design)
3. [Core Components](#core-components)
4. [Data Flow & Communication](#data-flow--communication)
5. [Implementation Key Points](#implementation-key-points)
6. [Security Considerations](#security-considerations)
7. [Workflow & Use Cases](#workflow--use-cases)
8. [Design Decisions & Trade-offs](#design-decisions--trade-offs)

---

## Technology Stack Overview

### Core Framework & API

| Technology | Version | Purpose | Rationale |
|---|---|---|---|
| **FastAPI** | ≥0.68.0 | Async web framework & REST API | Modern, high-performance Python framework with automatic API documentation and async support for non-blocking I/O |
| **Uvicorn** | ≥0.15.0 | ASGI application server | Lightweight, production-ready server for serving FastAPI applications with excellent async performance |
| **Pydantic** | ≥1.9.0 | Data validation & serialization | Type-safe request/response validation using Python type hints; ensures data integrity at API boundaries |

### Command Execution & Shell Integration

| Technology | Version | Purpose | Rationale |
|---|---|---|---|
| **Click** | ≥8.0.0 | CLI framework | Elegant command-line interface creation with groups, subcommands, and sophisticated option parsing |
| **Paramiko** | ≥2.8.0 | SSH protocol implementation | Pure Python SSH implementation enabling secure remote command execution with persistent sessions |
| **Python subprocess** | Built-in | Local command execution | Standard library for local command spawning with shell integration and pipe support |
| **ShLex** | Built-in | Shell command parsing | Proper shell tokenization for complex commands with pipes, redirects, and operators |

### Configuration & Environment Management

| Technology | Version | Purpose | Rationale |
|---|---|---|---|
| **python-dotenv** | ≥0.19.0 | Environment variable management | Convention-based .env file loading for secrets and configuration without code changes |
| **PyYAML** | ≥6.0 | YAML processing | Structured configuration and output formatting supporting complex nested structures |
| **Colorama** | ≥0.4.4 | Terminal color output | Cross-platform ANSI color support for enhanced CLI readability and visual feedback |

### HTTP & Data Handling

| Technology | Version | Purpose | Rationale |
|---|---|---|---|
| **python-multipart** | ≥0.0.5 | Multipart form data | HTTP form data parsing for file uploads and mixed content requests |
| **Protobuf** | 4.21.12 | Data serialization | Binary protocol for efficient data serialization where needed |

### Utilities

| Technology | Version | Purpose | Rationale |
|---|---|---|---|
| **deprecated** | ≥1.2.13 | Deprecation management | Graceful handling of legacy features with user warnings |

### CORS & Security

Built-in middleware for:
- **CORS (Cross-Origin Resource Sharing):** Enables frontend applications to interact with the API
- **Authentication:** Custom auth layer with JWT token support (currently disabled for local development)

---

## Architectural Design

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    Client Layer                                 │
│  (Web UI / CLI / SDK / AI Agent)                               │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼ HTTP/REST
┌─────────────────────────────────────────────────────────────────┐
│                    API Gateway Layer                            │
│  FastAPI Application (Port 7304)                               │
│  ├─ CORS Middleware                                            │
│  ├─ Authentication Middleware                                  │
│  └─ Exception Handling                                         │
└────────────────────┬────────────────────────────────────────────┘
                     │
        ┌────────────┼────────────┬───────────────┐
        ▼            ▼            ▼               ▼
    ┌────────┐  ┌────────┐  ┌────────┐  ┌──────────────┐
    │ Base   │  │ Meta   │  │Registry│  │ Session      │
    │Router  │  │Router  │  │Router  │  │ Router       │
    └────┬───┘  └───┬────┘  └───┬────┘  └───────┬──────┘
         │          │           │               │
         └──────────┼───────────┼───────────────┘
                    ▼
        ┌──────────────────────────────────┐
        │  Core Execution Layer            │
        │  ├─ CommandExecutor              │
        │  ├─ UnifiedExecutor              │
        │  ├─ RemoteExecutor               │
        │  └─ SessionManager               │
        └──────┬───────────────────────────┘
               │
    ┌──────────┼──────────┬─────────────────┐
    ▼          ▼          ▼                 ▼
┌────────┐ ┌────────┐ ┌────────┐  ┌──────────────┐
│ Local  │ │ Remote │ │ SSH    │  │ SSH Connection
│Command │ │Command │ │Client  │  │ Pool (persistent)
└────────┘ └────────┘ └────────┘  └──────────────┘
    │          │          │              │
    └──────────┼──────────┴──────────────┘
               ▼
    ┌──────────────────────────────────┐
    │  Metadata & Analysis Layer       │
    │  ├─ CommandMetadata              │
    │  ├─ CommandRegistry              │
    │  ├─ CommandDecision              │
    │  ├─ ModelHints                   │
    │  └─ ToolResultFormatter          │
    └──────────────────────────────────┘
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐
│Logging │ │Auditing│ │Session │
│ System │ │ System │ │History │
└────────┘ └────────┘ └────────┘
```

### Component Layers

#### 1. **API Gateway Layer** (api.py, routes.py)
- **Responsibility:** HTTP request handling, routing, and response formatting
- **Key Features:**
  - FastAPI application setup with CORS and middleware
  - OpenAPI schema customization
  - Global exception handling
  - Request/response validation via Pydantic models
- **Entry Points:**
  - `/api/run-command` - Execute commands
  - `/api/tool-result` - Process tool results with metadata
  - `/api/ssh-sessions` - Manage persistent SSH sessions
  - `/api/sessions/*` - Session lifecycle management

#### 2. **Core Execution Layer** (executor.py, remote.py, ssh.py)
- **Responsibility:** Command execution on local or remote systems
- **Key Components:**
  - **CommandExecutor:** Local command execution with metadata generation
  - **RemoteExecutor:** SSH-based remote execution with persistent connections
  - **UnifiedExecutor:** Unified interface for both local and remote execution
  - **SSHConnection:** Individual SSH connection with state preservation
  - **SSHConnectionPool:** Background service managing multiple persistent SSH connections

#### 3. **Metadata & Intelligence Layer** (metadata.py, registry.py, decision.py, hints.py)
- **Responsibility:** Command analysis, decision making, and AI optimization
- **Key Components:**
  - **CommandRegistry:** Central registry of known commands with metadata
  - **CommandMetadata:** Rich metadata generation including risk assessment and optimization
  - **CommandDecision:** Command analysis and alternative suggestions
  - **ModelHints:** AI/LLM optimization hints for better processing

#### 4. **State Management Layer** (session.py, pool.py)
- **Responsibility:** Session and connection state tracking
- **Key Features:**
  - Session lifecycle management with timeout handling
  - Command history tracking
  - Context preservation across commands
  - Background process tracking

#### 5. **Logging & Audit Layer** (logger.py, audit.py, telemetry.py)
- **Responsibility:** Comprehensive logging and audit trail
- **Key Features:**
  - Multi-level logging (API, system, command, decision)
  - Audit trail for security and compliance
  - Structured logging with correlation IDs
  - Log rotation and file management

---

## Core Components

### 1. CommandExecutor (executor.py)

**Purpose:** Primary interface for executing system commands with full metadata support.

**Key Methods:**
- `execute_command(command, cwd, timeout)` - Execute local command
- `execute(command, metadata_format)` - Execute with metadata generation
- `detect_command_type(command)` - Analyze command structure
- `_execute_with_retry(command, timeout)` - Resilient execution with retries

**Features:**
- Local command execution via subprocess
- Complex command support (pipes, redirects, operators)
- Timeout management
- Output cleaning and parsing
- Docker detection and host resolution
- Metadata enrichment on completion

**Execution Flow:**
```
Input Command
    │
    ├─→ Validate & Parse
    │
    ├─→ Detect Command Type (simple/complex/dangerous)
    │
    ├─→ Execute with Error Handling
    │
    ├─→ Capture stdout/stderr
    │
    ├─→ Generate Metadata (if enabled)
    │   ├─→ Registry lookup
    │   ├─→ Risk assessment
    │   ├─→ Output analysis
    │   ├─→ Decision analysis
    │   └─→ Model hints
    │
    └─→ Return Result with Metadata
```

### 2. RemoteExecutor & SSHConnection (remote.py, routes.py)

**Purpose:** Secure remote command execution via SSH with persistent connections.

**Key Classes:**
- **RemoteExecutor:** Context manager for SSH connections
- **SSHConnection:** Individual connection with state preservation
- **SSHConnectionPool:** Background service for connection pooling

**Features:**
- Password and key-based authentication
- Persistent SSH connections (avoid repeated handshakes)
- Directory state tracking across commands
- Environment variable injection
- Background process management
- Connection health monitoring with idle timeout
- Automatic reconnection on failure
- State persistence to disk for recovery

**State Preservation:**
```
Command Execution
    │
    ├─→ Track current directory
    ├─→ Track environment variables
    ├─→ Track background processes
    ├─→ Save state to JSON file
    │
    ├─→ On Disconnect
    │   └─→ Restore state on reconnect
    │
    └─→ Enable seamless session continuity
```

### 3. CommandRegistry (registry.py)

**Purpose:** Central repository of command metadata and knowledge.

**Key Methods:**
- `register_command(command, metadata)` - Register new command
- `get_command_metadata(command)` - Lookup command info
- `get_commands_by_category(category)` - Category-based search
- `get_related_commands(command)` - Find related commands

**Structure:**
```
CommandRegistry
    │
    ├─ commands: Dict[command_name → metadata]
    │   ├─ description
    │   ├─ common_options
    │   ├─ output_format
    │   ├─ category
    │   ├─ risk_level
    │   └─ related_commands
    │
    ├─ categories: Nested hierarchy
    │   ├─ filesystem
    │   │   ├─ navigation
    │   │   ├─ file_operations
    │   │   └─ search
    │   │
    │   ├─ system
    │   │   ├─ process_management
    │   │   ├─ user_management
    │   │   └─ resource_monitoring
    │   │
    │   └─ networking
    │       ├─ connection
    │       ├─ diagnostics
    │       └─ file_transfer
```

### 4. CommandMetadata (metadata.py)

**Purpose:** Rich metadata generation for command execution results.

**Generated Metadata:**
```json
{
  "command": "ls -la /tmp",
  "base_command": "ls",
  "execution_context": {
    "user": "ashish",
    "host": "localhost",
    "working_directory": "/home/ashish",
    "platform": "Linux",
    "timestamp": "2025-12-09T10:30:00Z"
  },
  "result": {
    "exit_code": 0,
    "success": true,
    "stdout_length": 1234,
    "stderr_length": 0,
    "execution_time_ms": 45
  },
  "registry": {
    "description": "List directory contents",
    "output_format": "tabular",
    "risk_level": "low"
  },
  "output_analysis": {
    "format_type": "tabular",
    "line_count": 42,
    "contains_errors": false
  },
  "user_experience": {
    "description": "List directory with all details",
    "estimated_time_ms": 50,
    "risk_level": "low",
    "status_emoji": "✅"
  },
  "next_steps": [
    "grep for specific files",
    "change directory",
    "view file contents"
  ]
}
```

### 5. CommandDecision (decision.py)

**Purpose:** Intelligent command analysis and decision support.

**Analysis Output:**
```json
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
      "reasoning": "Shorter syntax, requires elevated privileges in sudoers",
      "confidence": 0.92
    }
  ],
  "selection_confidence": 0.88,
  "risk_assessment": {
    "privilege_level": "elevated",
    "system_impact": "package_management",
    "requires_confirmation": true,
    "can_be_destructive": false
  }
}
```

### 6. ModelHints (hints.py)

**Purpose:** LLM/AI optimization hints for better output processing.

**Hint Categories:**
- **Parsing Strategy:** How to parse command output (tabular, line-by-line, JSON, etc.)
- **Token Efficiency:** Suggestions to reduce token consumption
- **Output Format:** Expected structure of command output
- **Confidence:** Parser confidence level
- **Command-Specific Hints:** Tailored suggestions for command type

### 7. SessionManager (session.py)

**Purpose:** Manage command execution sessions with context preservation.

**Session Lifecycle:**
```
Create Session
    │
    ├─ Initialize session with ID
    ├─ Set timeout (default 1 hour)
    ├─ Enable context tracking
    │
    ├─→ Execute Commands in Session
    │   ├─ Record each command
    │   ├─ Preserve working directory
    │   ├─ Track environment changes
    │   └─ Maintain command history
    │
    ├─ Query Session History
    │   ├─ Get command history
    │   ├─ Get suggestions
    │   └─ Analyze patterns
    │
    └─→ Session Expires
        └─ Cleanup and archive history
```

**Key Features:**
- Timeout-based session expiration
- Persistent history to disk
- Context-aware command suggestions
- Audit trail of all commands

---

## Data Flow & Communication

### 1. Local Command Execution Flow

```
HTTP Request (POST /api/run-command)
    │
    ├─ Validate CommandRequest
    │   ├─ Check command format
    │   ├─ Validate parameters
    │   └─ Check authentication
    │
    ├─ Route to Execution Layer
    │   └─ CommandExecutor.execute()
    │
    ├─ Local Execution
    │   ├─ Parse command
    │   ├─ Check for special patterns (cd, sudo, apt, etc.)
    │   ├─ Execute via subprocess
    │   ├─ Capture output (stdout, stderr)
    │   ├─ Track execution time
    │   └─ Clean output
    │
    ├─ Generate Metadata (conditional)
    │   ├─ Registry lookup
    │   ├─ Risk assessment
    │   ├─ Output analysis
    │   ├─ Decision analysis
    │   └─ Model hints
    │
    ├─ Record Session History
    │   └─ SessionManager.record_command()
    │
    └─ Return Response
        └─ CommandResponse (with optional metadata)
```

### 2. Remote Command Execution Flow

```
HTTP Request (POST /api/local-command-executor)
    │
    ├─ Parse Request
    │   ├─ Extract: host, port, username, password
    │   ├─ Extract: command, timeout
    │   └─ Extract: session_id
    │
    ├─ Connect or Reuse Connection
    │   │
    │   ├─ Check SSH Connection Pool
    │   │   └─ If exists: reuse persistent connection
    │   │
    │   └─ If not exists: Create new connection
    │       ├─ Establish SSH connection
    │       ├─ Load previous state (directory, env vars)
    │       ├─ Add to connection pool
    │       └─ Store connection reference
    │
    ├─ Execute Command
    │   ├─ Inject environment variables
    │   ├─ Execute on remote system
    │   ├─ Capture output
    │   └─ Save state (directory, env vars, processes)
    │
    ├─ Handle Special Cases
    │   ├─ CD commands: Update directory tracking
    │   ├─ sudo commands: Interactive password handling
    │   ├─ APT commands: Extended timeout, lock handling
    │   ├─ Docker/Complex: Multi-line execution
    │   └─ Background: Process PID tracking
    │
    ├─ Health Check & Cleanup
    │   ├─ Monitor connection health
    │   ├─ Clean expired processes
    │   ├─ Save state to persistent storage
    │   └─ Remove idle connections
    │
    └─ Return Response
        └─ CommandResponse with session info
```

### 3. Session-Based Context Flow

```
Client Request with session_id
    │
    ├─ SessionManager lookup
    │   └─ Retrieve existing session
    │
    ├─ Execute within Session Context
    │   ├─ Load working directory
    │   ├─ Load environment variables
    │   ├─ Load command history
    │   └─ Apply context to execution
    │
    ├─ Update Session
    │   ├─ Record new command
    │   ├─ Update working directory
    │   ├─ Update environment
    │   ├─ Update history
    │   └─ Persist to disk
    │
    └─ Return Response with Updated Context
        └─ Include command history, suggestions, etc.
```

### 4. API Communication Patterns

**Request Format:**
```json
{
  "command": "npm start",
  "remote": false,
  "session_id": "abc-123",
  "timeout": 30
}
```

**Response Format:**
```json
{
  "success": true,
  "exit_code": 0,
  "stdout": "...",
  "stderr": "",
  "command": "npm start",
  "execution_time": 2.34,
  "metadata": {
    "command": "npm start",
    "result": {...},
    "registry": {...},
    "decision": {...},
    "model_hints": {...}
  }
}
```

---

## Implementation Key Points

### 1. Command Structure & Parsing Logic

**Command Types Detected:**

| Type | Pattern | Handling |
|------|---------|----------|
| **Simple** | `ls` `whoami` `pwd` | Direct execution |
| **Arguments** | `ls -la /tmp` | Parsed and passed to subprocess |
| **Pipes** | `cat file \| grep text` | Shell execution via `/bin/bash -c` |
| **Redirects** | `echo text > file` | Shell execution for file operations |
| **Operators** | `cmd1 && cmd2` `cmd1 \|\| cmd2` | Shell chaining |
| **sudo** | `sudo command` | Special handling with password injection |
| **cd** | `cd /path` | Custom handling to persist state |
| **apt/apt-get** | `apt update` | Extended timeout, lock detection |
| **docker** | `docker run ...` | exec_command for better output capture |
| **heredoc** | `cat <<EOF` | Multi-line input handling |

**Parsing Strategy:**
```python
def detect_command_type(command: str) -> dict:
    # Check for shell operators
    has_pipe = '|' in command
    has_redirect = '>' in command or '<' in command
    has_chain = '&&' in command or '||' in command or ';' in command
    has_sudo = 'sudo' in command
    
    # Special command handlers
    if command.startswith('cd '):
        return {'type': 'directory_change', 'needs_special_handling': True}
    
    if any(apt_cmd in command for apt_cmd in ['apt ', 'apt-get ']):
        return {'type': 'package_manager', 'timeout_multiplier': 3}
    
    # Complex commands need shell interpretation
    if any([has_pipe, has_redirect, has_chain]):
        return {'type': 'complex', 'execute_via': 'shell'}
    
    # Simple commands can use subprocess directly
    return {'type': 'simple', 'execute_via': 'direct'}
```

### 2. Configuration Handling

**Configuration Sources (Priority Order):**

1. **Environment Variables** (highest priority)
   - Override all other sources
   - Source: System environment or `.env` file
   - Examples: `API_SERVER_URL`, `FASTAPI_PORT`, `DOCKER_HOST_IP`

2. **Settings Object** (config/settings.py)
   - Loaded from environment via `dotenv`
   - Pydantic-based validation
   - Default values for missing variables

3. **Runtime Configuration**
   - CLI options override defaults
   - Session-specific settings
   - Connection-specific parameters

**Key Configuration:**
```python
# .env file structure
JWT_ALGORITHM=RS256
JWT_AUDIENCE=account
ENV=development
BYPASS_AUTH=false
FORCE_LOCAL_EXECUTION=false
DISABLE_REMOTE=false
API_SERVER_URL=http://localhost:7304
FASTAPI_PORT=7304
DOCKER_HOST_IP=192.168.1.X
```

### 3. Error Handling Strategy

**Layered Error Handling:**

```
Level 1: Validation Errors
├─ Pydantic validation failures
├─ Missing required fields
└─ Type mismatches
    └─→ Return HTTP 422

Level 2: Authentication/Authorization
├─ Failed auth checks
├─ Insufficient permissions
└─ Token issues
    └─→ Return HTTP 401/403

Level 3: Execution Errors
├─ SSH connection failures
├─ Command execution timeout
├─ Non-zero exit codes
└─ Permission denied
    └─→ Return HTTP 200 (successful API call) with success=false

Level 4: Unhandled Exceptions
├─ Internal server errors
├─ Unexpected failures
└─ System resource exhaustion
    └─→ Return HTTP 500 with traceback
```

**Retry Logic:**
```python
def _execute_with_retry(command: str, timeout: int, max_retries: int = 2):
    for attempt in range(max_retries):
        try:
            result = execute_command(command, timeout)
            return result
        
        except TimeoutError as e:
            # Retry on timeout (transient)
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
                continue
            raise
        
        except (ConnectionError, BrokenPipeError) as e:
            # Retry on connection issues (transient)
            if attempt < max_retries - 1:
                reconnect()
                continue
            raise
        
        except (PermissionError, FileNotFoundError) as e:
            # Don't retry permanent failures
            raise
```

### 4. Security Considerations

#### Authentication
- **Current Status:** Disabled for local development
- **Implementation:** Auth middleware in `auth.py`
- **Future Support:** 
  - JWT tokens with RS256 algorithm
  - Role-based access control (RBAC)
  - Per-endpoint permission checks

#### Command Execution Security

**Dangerous Command Detection:**
```python
dangerous_patterns = [
    r'rm\s+-rf\s+/',          # Recursive delete of root
    r'dd\s+if=.*of=/dev/',    # Direct disk write
    r'mkfs',                    # Format filesystem
    r':(){ :|:& };:',          # Fork bomb
    r'>\s*/dev/sda',           # Disk overwrite
]

def is_dangerous_command(command: str) -> bool:
    for pattern in dangerous_patterns:
        if re.search(pattern, command):
            return True
    return False
```

**Privilege Escalation Controls:**
- Detect `sudo` and `su` commands
- Optional confirmation for elevated operations
- Audit logging of privileged commands
- User impersonation prevention

#### Environment Variable Validation
```python
def _is_safe_env_value(self, value: str) -> bool:
    """Validate environment variable values are safe"""
    # Reject values containing dangerous command operators
    dangerous_patterns = [
        r'[;&|`]',           # Command separators
        r'\$\(',             # Command substitution
        r'>\s*[/\w]',        # Output redirection
        r'<\s*[/\w]',        # Input redirection
    ]
    
    for pattern in dangerous_patterns:
        if re.search(pattern, value):
            return False
    
    # Allow common safe patterns
    safe_patterns = [
        r'^[a-zA-Z0-9_/:~.\-\s]+$',              # Alphanumeric, paths
        r'^\$[A-Za-z_][A-Za-z0-9_]*$',           # Variable references
        r'^[a-zA-Z]+://[^\s]+$',                 # URLs
    ]
    
    return any(re.match(pattern, value) for pattern in safe_patterns)
```

#### SSH Security
- Password stored in memory only (not persisted)
- Key-based authentication support
- Host key verification via `AutoAddPolicy`
- Timeout on unresponsive connections
- Connection health monitoring

#### Audit Logging
```python
# Every significant operation is logged:
- User identity
- Command executed
- Remote host (if applicable)
- Exit code
- Execution time
- Timestamp
- Session ID (if applicable)
```

### 5. Extensibility & Plugin Design

**Registry-Based Extensibility:**

The `CommandRegistry` allows adding new command metadata:

```python
# Register new command
registry = CommandRegistry()
registry.register_command('custom-tool', {
    'description': 'Custom tool description',
    'category': 'custom.tools',
    'output_format': 'json',
    'risk_level': 'low',
    'common_options': [
        {'flag': '-v', 'description': 'Verbose output'},
        {'flag': '--json', 'description': 'JSON output'}
    ],
    'related_commands': ['another-tool']
})
```

**Custom Execution Handlers:**

The executor supports custom handlers for special commands:

```python
# Special handling for custom commands
if 'custom-tool' in command:
    return self._handle_custom_tool_command(command, timeout)

# Future: plugin system for third-party handlers
plugins = load_plugins()
for handler in plugins:
    if handler.matches(command):
        return handler.execute(command, context)
```

**Metadata Enhancement Pipeline:**

Metadata is generated through multiple stages, each extensible:

1. **Registry Enrichment:** Lookup command in registry
2. **Output Analysis:** Analyze actual output structure
3. **Decision Analysis:** Analyze command and alternatives
4. **Model Hints:** Generate LLM optimization hints
5. **Next Steps:** Suggest follow-up commands

### 6. State Persistence

**Session State Storage:**

Sessions and SSH connections maintain state across calls:

```
~/.local/share/cli-tool/
├── sessions/
│   ├── session-id-1/
│   │   ├── history.json      # Command history
│   │   ├── context.json      # Execution context
│   │   └── metadata.json     # Collected metadata
│   └── session-id-2/
│       └── ...
│
└── ssh-states/
    ├── ssh_state_session-key-1.json  # SSH connection state
    │   ├── current_directory
    │   ├── environment_vars
    │   └── background_processes
    └── ...
```

**Atomic State Writes:**

```python
def _save_state(self):
    """FIXED: Save current state to disk with atomic write"""
    state = {
        'current_directory': self.current_directory,
        'environment_vars': self.environment_vars,
        'background_processes': self.background_processes,
        'last_saved': datetime.utcnow().isoformat()
    }
    
    # Atomic write to prevent corruption
    temp_file = f"{self.state_file}.tmp"
    try:
        with open(temp_file, 'w') as f:
            json.dump(state, f)
        # Atomic rename
        os.rename(temp_file, self.state_file)
    except Exception as e:
        logger.error(f"Failed to save state: {e}")
```

---

## Security Considerations

### 1. Input Validation

All external inputs are validated at multiple levels:

1. **API Level:** Pydantic model validation
   ```python
   class CommandRequest(BaseModel):
       command: str = Field(..., description="The terminal command")
       remote: bool = Field(False)
       host: Optional[str] = Field(None)
       timeout: Optional[int] = Field(30, ge=1, le=3600)  # Range validation
   ```

2. **Execution Level:** Command pattern analysis
   ```python
   # Detect and handle special patterns
   if any(dangerous_pattern in command for dangerous_pattern in DANGEROUS_PATTERNS):
       logger.warning(f"Potentially dangerous command: {command}")
       if not ALLOW_DANGEROUS_COMMANDS:
           raise ValueError("Command blocked for security")
   ```

3. **SSH Level:** Connection validation
   ```python
   # Validate SSH connection parameters
   if not host or not username:
       raise ValueError("SSH requires host and username")
   ```

### 2. Authentication & Authorization

**Current Implementation (Development):**
- Authentication disabled for local development
- Dummy user returned on all requests
- Can be enabled via `BYPASS_AUTH` environment variable

**Future Implementation:**
- JWT-based authentication
- Role-based access control (RBAC)
- Command-level permissions
- Resource quotas per user

### 3. Secrets Management

**Password Handling:**
- Passwords accepted only via request body (not URL)
- Stored in memory for SSH connection
- Not logged or persisted
- Cleared after connection closes

**Key File Management:**
- Path validated before use
- File permissions checked
- Content not logged
- Private key format validated

### 4. Session Security

**Session Isolation:**
- Each session has unique ID
- Sessions tied to user identity
- Session timeout after inactivity
- History not shared between sessions

**Session Expiration:**
```python
def is_expired(self) -> bool:
    return time.time() - self.last_used_at > self.timeout

# Cleanup loop runs periodically
def _cleanup_loop(self):
    while self.running:
        expired_sessions = [s for s in sessions if s.is_expired()]
        for session in expired_sessions:
            session.close()
            logger.info(f"Cleaned up expired session: {session.id}")
        time.sleep(self.cleanup_interval)
```

### 5. Audit Trail

Comprehensive logging of all operations:

```python
# Audit logging of SSH connections
audit_logger.log_connection(
    action="connect",
    host=self.host,
    username=self.username,
    success=True
)

# Audit logging of commands
audit_logger.log_command(
    command=command,
    session_id=session_id,
    exit_code=exit_code,
    success=exit_code == 0,
    execution_time=elapsed
)

# Audit logging of system events
audit_logger.log_system_event(
    event_type="ssh_connect",
    details={
        "host": host,
        "auth_type": auth_type,
        "connection_time": f"{elapsed:.3f}s"
    }
)
```

### 6. CORS & Web Security

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Configure for production
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
    expose_headers=["*"]
)
```

**Production Recommendations:**
- Restrict `allow_origins` to specific domains
- Use HTTPS in production
- Implement CSRF protection
- Add rate limiting
- Implement API key authentication

---

## Workflow & Use Cases

### Use Case 1: Simple Local Command Execution

**Scenario:** User wants to list files in a directory from a web UI.

**Workflow:**

```
1. Frontend sends HTTP request
   POST /api/run-command
   {
     "command": "ls -la /home/user/project"
   }

2. API validates request
   - Check command format
   - Verify authentication
   - Apply default timeout (30s)

3. Local execution
   - Detect: simple command with arguments
   - Execute: subprocess.run(['ls', '-la', '/home/user/project'])
   - Capture: stdout, stderr, exit_code
   - Time: 0.045 seconds

4. Metadata generation
   - Registry lookup: found 'ls' command
   - Output analysis: Identified as tabular format
   - Risk assessment: low risk
   - Suggestions: grep, find, cd, etc.

5. Return response
   {
     "success": true,
     "exit_code": 0,
     "stdout": "drwxr-xr-x  12 user user  4096 Dec  9 10:30 .",
     "stderr": "",
     "metadata": {
       "registry": {...},
       "output_analysis": {...},
       "next_steps": [...]
     }
   }
```

### Use Case 2: Remote Development Server Management

**Scenario:** AI Agent wants to start an npm development server on a remote machine and monitor its output.

**Workflow:**

```
1. Create persistent SSH session
   POST /api/create-session
   {
     "client_info": {"tool": "ai_agent"}
   }
   Response: {"session_id": "abc-123"}

2. First remote command (with SSH connection)
   POST /api/local-command-executor
   {
     "command": "pwd",
     "remote": true,
     "host": "dev.example.com",
     "username": "deploy",
     "password": "***",
     "session_id": "abc-123"
   }
   
   - Create SSH connection
   - Add to connection pool
   - Execute command
   - Save state (directory: /home/deploy)

3. Navigate to project directory
   POST /api/local-command-executor
   {
     "command": "cd /home/deploy/myapp",
     "session_id": "abc-123"
   }
   
   - Use persistent SSH connection
   - Special CD handling (update state)
   - Save new directory state

4. Start dev server (background)
   POST /api/local-command-executor
   {
     "command": "npm start",
     "session_id": "abc-123",
     "timeout": 60
   }
   
   - Execute in background
   - Track background process
   - Return PID for monitoring

5. Query session info
   GET /api/session/abc-123
   
   Response:
   {
     "session_id": "abc-123",
     "current_directory": "/home/deploy/myapp",
     "command_count": 3,
     "active_task": "npm_server",
     "command_history": [...]
   }

6. Monitor logs
   GET /api/dev-server-logs/abc-123?lines=50
   
   Response: Recent 50 lines of npm output

7. Stop dev server
   POST /api/process-manager
   {
     "action": "kill",
     "session_key": "abc-123",
     "pid": 12345
   }
```

### Use Case 3: Batch Command Execution with Context

**Scenario:** System setup automation requiring multiple commands with shared context.

**Workflow:**

```
1. Create session for batch operations
   POST /api/create-session

2. Execute setup commands in sequence
   
   a. Update system packages
      POST /api/run-command
      {
        "command": "sudo apt-get update",
        "session_id": "batch-123",
        "timeout": 120
      }
      
      Handles:
      - sudo password prompt
      - APT lock checking
      - Extended timeout (120s)
   
   b. Install dependencies
      POST /api/run-command
      {
        "command": "sudo apt-get install -y nodejs npm",
        "session_id": "batch-123",
        "timeout": 120
      }
   
   c. Install project dependencies
      POST /api/run-command
      {
        "command": "npm install",
        "session_id": "batch-123",
        "timeout": 300
      }
      
      Context: Still in project directory from previous cd

3. Retrieve full batch history
   GET /api/session/batch-123/history
   
   Response: All commands with metadata and results

4. Analyze execution patterns
   - Total time: 4 minutes 32 seconds
   - Success rate: 100%
   - Most common commands: sudo, npm, apt
```

### Use Case 4: Interactive Troubleshooting with AI

**Scenario:** AI assistant helping user debug a failing application.

**Workflow:**

```
User: "Why is my Node server not starting?"

1. AI suggests diagnostics
   Command: "netstat -tlnp | grep 3000"
   
2. User (via AI) executes
   POST /api/run-command
   {
     "command": "netstat -tlnp | grep 3000",
     "session_id": "debug-123"
   }
   
   Response shows port not in use

3. AI suggests checking logs
   Command: "npm start 2>&1 | head -20"
   
   Response: "Error: ENOENT: no such file or directory, open '/app/config.json'"

4. AI suggests checking file
   Command: "ls -la /app/*.json"
   
5. AI suggests solution
   "Configuration file missing. Create it with: npm run init:config"

6. User confirms and executes
   Command: "npm run init:config"
   
7. Retry server start
   Command: "npm start"
   
   Response shows successful startup

AI maintains full context throughout session, using command history
and metadata to provide intelligent suggestions.
```

---

## Design Decisions & Trade-offs

### 1. FastAPI vs Other Frameworks

**Choice:** FastAPI

**Rationale:**
- ✅ Async/await for non-blocking I/O
- ✅ Automatic OpenAPI documentation
- ✅ Built-in data validation (Pydantic)
- ✅ Fast performance (near-native speed)
- ✅ Modern Python (3.6+)

**Trade-off:**
- ❌ Smaller ecosystem than Django/Flask
- ❌ Less mature (newer framework)
- ✅ But excellent for APIs like this

### 2. Persistent SSH Connections vs One-Shot

**Choice:** Persistent connection pool with state preservation

**Rationale:**
- ✅ Significant performance improvement (no SSH handshake per command)
- ✅ State preservation (working directory, env vars)
- ✅ Background process tracking
- ✅ Better user experience for long-running tasks

**Trade-off:**
- ❌ More complex state management
- ❌ Memory overhead for idle connections
- ❌ Timeout and cleanup logic needed
- ✅ Solves real user pain points

### 3. Subprocess vs Custom Shell Parser

**Choice:** Use subprocess with shell operators for complex commands

**Rationale:**
- ✅ Reuses battle-tested shell parser (bash)
- ✅ Supports all shell features (pipes, redirects, operators)
- ✅ User-familiar syntax
- ✅ Easier than building custom parser

**Trade-off:**
- ❌ Security risks with user input (mitigated by validation)
- ❌ Requires careful escaping
- ✅ Essential for real-world usage

### 4. Metadata Generation Strategy

**Choice:** Optional metadata with multiple formats

**Rationale:**
- ✅ No performance overhead if not requested
- ✅ Flexible for different clients (full, summary, minimal)
- ✅ AI systems can request optimal format
- ✅ Backward compatible

**Trade-off:**
- ❌ Multiple code paths for different formats
- ✅ Worth it for flexibility

### 5. Registry-Based Extensibility

**Choice:** Centralized command registry over per-execution analysis

**Rationale:**
- ✅ Pre-computed metadata for common commands
- ✅ Fast lookup
- ✅ Consistent information
- ✅ Allows custom metadata registration

**Trade-off:**
- ❌ Must maintain registry
- ❌ Unknown commands have limited metadata
- ✅ Fallback to dynamic analysis for unknown commands

### 6. State Persistence

**Choice:** Atomic JSON file writes for SSH connection state

**Rationale:**
- ✅ Simple, human-readable format
- ✅ Easy debugging and inspection
- ✅ Atomic writes prevent corruption
- ✅ No external dependencies

**Trade-off:**
- ❌ File I/O overhead
- ❌ Not suitable for very high frequency updates
- ✅ Sufficient for command execution pace

### 7. Error Handling Philosophy

**Choice:** Distinguish API errors vs command execution errors

**Rationale:**
- ✅ API errors (400, 401, 500) for system-level issues
- ✅ Command errors (exit code) return HTTP 200 with success=false
- ✅ Clear distinction helps clients handle appropriately
- ✅ Aligns with actual vs logical error distinction

**Trade-off:**
- ❌ Less intuitive than failing on command errors
- ✅ More correct semantically

### 8. Timeout Management

**Choice:** Adaptive timeout based on command type

**Rationale:**
- ✅ Most commands finish in <5s (default 30s)
- ✅ Package managers need longer (e.g., apt = 120s)
- ✅ Docker commands need even longer
- ✅ Server start commands need even more

**Trade-off:**
- ❌ Must maintain timeout rules per command type
- ✅ Improves reliability without manual tweaking

---

## Deployment Architecture

### Docker-Based Deployment

The application is containerized for consistent deployment:

**Dockerfile Strategy:**
- Base: `python:3.11-slim` (minimal footprint)
- Dependencies: pip install from `requirements.txt`
- Entrypoint: `uvicorn` server on port 7304
- Volume mounts: logs, sessions directories

**docker-compose.yaml:**
- FastAPI service on port 7304
- Volume persistence for logs and sessions
- Environment variable injection
- Network configuration for cross-container communication

### Running the Application

**Development:**
```bash
# Without Docker
python -m uvicorn api:app --host 0.0.0.0 --port 7304 --reload

# With Docker
docker-compose up --build
```

**Production:**
```bash
# With Docker
docker-compose up -d

# Monitoring
docker logs -f cli_tool_app
docker stats
```

---

## Summary

The CLI Tool is architected as a modern, async-first command execution platform with these key characteristics:

1. **Layered Architecture:** Clear separation of concerns across API, execution, metadata, and state layers
2. **Flexible Execution:** Support for both local and remote (SSH) command execution
3. **Rich Metadata:** Comprehensive command analysis for AI/LLM systems
4. **Session Management:** Context-aware execution with persistent state
5. **Security First:** Multiple layers of validation, authentication, and audit logging
6. **Extensibility:** Registry-based system for adding new command support
7. **Reliability:** Retry logic, timeout management, error recovery
8. **Observability:** Detailed logging, audit trails, and telemetry

This design enables the CLI Tool to serve as a robust backend for AI agents, web UIs, and programmatic access to system commands while maintaining security, reliability, and performance.

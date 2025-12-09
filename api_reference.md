# CLI Tool - API Reference Guide

## Overview

This document provides detailed API endpoint reference for the CLI Tool. All endpoints require authentication (except `/health`) and return structured responses.

## Base URL

```
http://localhost:7304/api
```

## Authentication

**Current Status:** Disabled for local development

**Future Implementation:** JWT Bearer token required in Authorization header:
```
Authorization: Bearer <jwt-token>
```

## Response Format

All responses follow this structure:

```json
{
  "success": true|false,
  "exit_code": 0|non-zero,
  "stdout": "command output",
  "stderr": "error output if any",
  "command": "executed command",
  "execution_time": 0.123,
  "metadata": {...},
  "model_hints": {...},
  "decision": {...}
}
```

## Error Response Format

```json
{
  "error": "Error message",
  "detail": "Additional details",
  "traceback": "Full traceback (debug mode only)"
}
```

---

## Endpoints

### 1. Execute Local Command

**Endpoint:** `POST /run-command`

**Purpose:** Execute a command on the local system.

**Request:**
```json
{
  "command": "ls -la /home/user",
  "timeout": 30,
  "session_id": "optional-session-id"
}
```

**Parameters:**
| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `command` | string | Yes | - | Command to execute |
| `remote` | boolean | No | false | Execute on remote host |
| `timeout` | integer | No | 30 | Timeout in seconds (1-3600) |
| `session_id` | string | No | - | Session ID for context |

**Response:**
```json
{
  "success": true,
  "exit_code": 0,
  "stdout": "total 42\ndrwxr-xr-x...",
  "stderr": "",
  "command": "ls -la /home/user",
  "execution_time": 0.045
}
```

**Status Codes:**
- `200` - Command executed successfully
- `400` - Invalid request
- `401` - Unauthorized
- `422` - Validation error
- `500` - Internal server error

**Examples:**

```bash
# Simple command
curl -X POST http://localhost:7304/api/run-command \
  -H "Content-Type: application/json" \
  -d '{"command": "pwd"}'

# With timeout
curl -X POST http://localhost:7304/api/run-command \
  -H "Content-Type: application/json" \
  -d '{"command": "npm start", "timeout": 60}'

# With session
curl -X POST http://localhost:7304/api/run-command \
  -H "Content-Type: application/json" \
  -d '{"command": "ls", "session_id": "abc-123"}'
```

---

### 2. Execute Remote Command

**Endpoint:** `POST /local-command-executor`

**Purpose:** Execute a command on a remote machine via SSH.

**Request:**
```json
{
  "command": "ls -la",
  "remote": true,
  "host": "server.example.com",
  "port": 22,
  "username": "deploy",
  "password": "password123",
  "timeout": 30,
  "session_id": "optional-session-id"
}
```

**Parameters:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `command` | string | Yes | Command to execute remotely |
| `remote` | boolean | Yes | Must be true for remote execution |
| `host` | string | Yes | Remote hostname or IP |
| `port` | integer | No | SSH port (default 22) |
| `username` | string | Yes | SSH username |
| `password` | string | Yes | SSH password |
| `timeout` | integer | No | Command timeout in seconds |
| `session_id` | string | No | Session ID for persistent connection |

**Response:**
```json
{
  "success": true,
  "exit_code": 0,
  "stdout": "command output",
  "stderr": "",
  "command": "ls -la",
  "execution_time": 0.234
}
```

**Special Handling:**
- **cd commands:** Directory changes persist across commands in same session
- **sudo commands:** Password automatically injected
- **APT commands:** Timeout automatically extended to 120s
- **Docker commands:** Better output capture via exec_command

**Examples:**

```bash
# Basic remote execution
curl -X POST http://localhost:7304/api/local-command-executor \
  -H "Content-Type: application/json" \
  -d '{
    "command": "pwd",
    "remote": true,
    "host": "server.example.com",
    "username": "deploy",
    "password": "secret"
  }'

# Persistent session
curl -X POST http://localhost:7304/api/local-command-executor \
  -H "Content-Type: application/json" \
  -d '{
    "command": "npm start",
    "remote": true,
    "host": "dev.example.com",
    "username": "deploy",
    "password": "secret",
    "session_id": "dev-session",
    "timeout": 120
  }'
```

---

### 3. Get SSH Sessions

**Endpoint:** `GET /ssh-sessions`

**Purpose:** List all active SSH sessions in the connection pool.

**Response:**
```json
{
  "active_sessions": 2,
  "sessions": [
    {
      "session_key": "key-1",
      "host": "server1.example.com",
      "username": "deploy",
      "connected": true,
      "created_at": "2025-12-09T10:00:00Z",
      "last_used": "2025-12-09T10:05:23Z",
      "idle_time_seconds": 45
    },
    {
      "session_key": "key-2",
      "host": "server2.example.com",
      "username": "deploy",
      "connected": true,
      "created_at": "2025-12-09T10:02:00Z",
      "last_used": "2025-12-09T10:05:30Z",
      "idle_time_seconds": 0
    }
  ]
}
```

**Examples:**

```bash
curl http://localhost:7304/api/ssh-sessions
```

---

### 4. Close SSH Session

**Endpoint:** `DELETE /ssh-sessions/{session_key}`

**Purpose:** Close a specific SSH session and cleanup resources.

**Response:**
```json
{
  "success": true,
  "message": "SSH session closed successfully"
}
```

**Examples:**

```bash
curl -X DELETE http://localhost:7304/api/ssh-sessions/key-1
```

---

### 5. Tool Result Processing

**Endpoint:** `POST /tool-result`

**Purpose:** Process command results with advanced metadata formatting.

**Request:**
```json
{
  "command": "ls -la /home",
  "conversation_id": "conv-123",
  "conversation_message_id": "msg-456",
  "user_id": "user-789"
}
```

**Response:**
```json
{
  "success": true,
  "exit_code": 0,
  "stdout": "...",
  "metadata": {
    "command": "ls -la /home",
    "result": {...},
    "registry": {...},
    "decision": {...},
    "model_hints": {...}
  }
}
```

---

### 6. Create Session

**Endpoint:** `POST /create-session`

**Purpose:** Create a new execution session for context-aware command execution.

**Request:**
```json
{
  "session_id": "optional-specific-id",
  "client_info": {
    "tool": "ai_agent",
    "version": "1.0"
  }
}
```

**Response:**
```json
{
  "session_id": "abc-123-def-456",
  "created": true,
  "created_at": 1702118400.0,
  "timeout": 3600
}
```

**Examples:**

```bash
curl -X POST http://localhost:7304/api/create-session \
  -H "Content-Type: application/json" \
  -d '{
    "client_info": {
      "tool": "my_ai_agent",
      "version": "1.0"
    }
  }'
```

---

### 7. Get Session Info

**Endpoint:** `GET /session/{session_id}`

**Purpose:** Get information about a specific session.

**Response:**
```json
{
  "session_id": "abc-123",
  "created_at": 1702118400.0,
  "last_activity": 1702118450.0,
  "command_count": 5,
  "active_task": "npm_start",
  "current_directory": "/home/user/project",
  "context_enabled": true
}
```

**Examples:**

```bash
curl http://localhost:7304/api/session/abc-123
```

---

### 8. Get Session History

**Endpoint:** `GET /session/{session_id}/history`

**Purpose:** Get command history for a session.

**Parameters:**
| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `limit` | integer | 20 | Number of historical commands to retrieve |

**Response:**
```json
[
  {
    "command": "pwd",
    "timestamp": 1702118400.0,
    "exit_code": 0,
    "success": true,
    "execution_time": 0.023,
    "metadata": {...}
  },
  {
    "command": "ls -la",
    "timestamp": 1702118405.0,
    "exit_code": 0,
    "success": true,
    "execution_time": 0.045,
    "metadata": {...}
  }
]
```

**Examples:**

```bash
# Get last 20 commands
curl http://localhost:7304/api/session/abc-123/history

# Get last 50 commands
curl http://localhost:7304/api/session/abc-123/history?limit=50
```

---

### 9. Get Command Suggestions

**Endpoint:** `GET /session/{session_id}/suggestions`

**Purpose:** Get AI-suggested next commands based on session context.

**Parameters:**
| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `count` | integer | 5 | Number of suggestions to return |

**Response:**
```json
[
  {
    "command": "npm start",
    "reasoning": "You have package.json in current directory",
    "confidence": 0.95
  },
  {
    "command": "npm test",
    "reasoning": "Common next step after npm install",
    "confidence": 0.85
  },
  {
    "command": "git status",
    "reasoning": "Git repository detected in working directory",
    "confidence": 0.82
  }
]
```

**Examples:**

```bash
curl http://localhost:7304/api/session/abc-123/suggestions?count=10
```

---

### 10. Close Session

**Endpoint:** `DELETE /session/{session_id}`

**Purpose:** Close a session and cleanup resources.

**Response:**
```json
{
  "success": true,
  "message": "Session closed successfully"
}
```

**Examples:**

```bash
curl -X DELETE http://localhost:7304/api/session/abc-123
```

---

### 11. Process Management

**Endpoint:** `POST /process-manager`

**Purpose:** Manage background processes in SSH sessions.

**Request:**
```json
{
  "action": "list|check|kill",
  "session_key": "session-key-1",
  "pid": 12345
}
```

**Parameters:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action` | string | Yes | Operation: "list", "check", or "kill" |
| `session_key` | string | Yes | SSH session key |
| `pid` | integer | For "check"/"kill" | Process ID (required for check/kill) |

**Response (list):**
```json
{
  "processes": [
    {
      "pid": 12345,
      "command": "npm start",
      "started_at": "2025-12-09T10:00:00Z",
      "status": "running"
    }
  ]
}
```

**Response (check):**
```json
{
  "pid": 12345,
  "status": "running",
  "uptime_seconds": 300
}
```

**Response (kill):**
```json
{
  "success": true,
  "pid": 12345,
  "message": "Process killed successfully"
}
```

---

### 12. Health Check

**Endpoint:** `GET /health`

**Purpose:** Check API health status (no authentication required).

**Response:**
```json
{
  "status": "healthy",
  "version": "1.0.0"
}
```

**Examples:**

```bash
curl http://localhost:7304/api/health
```

---

### 13. Get Registry

**Endpoint:** `GET /registry`

**Purpose:** Get registered command metadata.

**Response:**
```json
{
  "total_commands": 145,
  "categories": ["filesystem", "system", "networking"],
  "commands": {
    "ls": {
      "description": "List directory contents",
      "category": "filesystem.content",
      "output_format": "tabular",
      "risk_level": "low",
      "common_options": [
        {"flag": "-l", "description": "Long format"},
        {"flag": "-a", "description": "Show hidden files"}
      ]
    }
  }
}
```

---

## Common Response Patterns

### Success Response
```json
{
  "success": true,
  "exit_code": 0,
  "stdout": "output",
  "stderr": "",
  "command": "ls",
  "execution_time": 0.045,
  "metadata": {...}
}
```

### Command Failed (Exit Code != 0)
```json
{
  "success": false,
  "exit_code": 127,
  "stdout": "",
  "stderr": "command not found",
  "command": "nonexistent",
  "execution_time": 0.012
}
```

### API Error
```json
{
  "error": "Invalid request",
  "detail": "Command field is required",
  "traceback": "..."
}
```

---

## Rate Limiting (Future)

Currently, no rate limiting is implemented. Production deployment should include:

- Per-user rate limits
- Per-IP rate limits
- Per-endpoint limits
- Burst allowances for specific operations

---

## Examples

### Example 1: Simple File Listing

```bash
curl -X POST http://localhost:7304/api/run-command \
  -H "Content-Type: application/json" \
  -d '{"command": "ls -la /tmp"}'
```

**Response:**
```json
{
  "success": true,
  "exit_code": 0,
  "stdout": "total 128\ndrwxrwxrwt 15 root root...",
  "stderr": "",
  "command": "ls -la /tmp",
  "execution_time": 0.034
}
```

### Example 2: Remote Server Deployment

```bash
# 1. Create session
SESSION=$(curl -X POST http://localhost:7304/api/create-session \
  -H "Content-Type: application/json" \
  -d '{"client_info":{"tool":"deployer"}}' | jq -r '.session_id')

# 2. SSH to server
curl -X POST http://localhost:7304/api/local-command-executor \
  -H "Content-Type: application/json" \
  -d "{
    \"command\": \"cd /app && git pull\",
    \"remote\": true,
    \"host\": \"server.example.com\",
    \"username\": \"deploy\",
    \"password\": \"secret\",
    \"session_id\": \"$SESSION\"
  }"

# 3. Restart service
curl -X POST http://localhost:7304/api/local-command-executor \
  -H "Content-Type: application/json" \
  -d "{
    \"command\": \"sudo systemctl restart myapp\",
    \"remote\": true,
    \"host\": \"server.example.com\",
    \"username\": \"deploy\",
    \"password\": \"secret\",
    \"session_id\": \"$SESSION\"
  }"

# 4. Check status
curl -X POST http://localhost:7304/api/local-command-executor \
  -H "Content-Type: application/json" \
  -d "{
    \"command\": \"systemctl status myapp\",
    \"remote\": true,
    \"host\": \"server.example.com\",
    \"username\": \"deploy\",
    \"password\": \"secret\",
    \"session_id\": \"$SESSION\"
  }"

# 5. Get full history
curl http://localhost:7304/api/session/$SESSION/history
```

### Example 3: Long-Running Dev Server

```bash
# Create session
curl -X POST http://localhost:7304/api/create-session \
  -H "Content-Type: application/json" \
  -d '{"client_info":{"tool":"dev_ui"}}' > session.json

SESSION=$(jq -r '.session_id' session.json)

# Start dev server with extended timeout
curl -X POST http://localhost:7304/api/run-command \
  -H "Content-Type: application/json" \
  -d "{
    \"command\": \"npm start\",
    \"timeout\": 3600,
    \"session_id\": \"$SESSION\"
  }"

# Poll for session info
watch "curl http://localhost:7304/api/session/$SESSION"

# Get recent output
curl http://localhost:7304/api/dev-server-logs/$SESSION?lines=50
```

---

## Implementation Notes

### Command Execution Context

When executing commands in a session:

1. Commands executed in sequence maintain state
2. Working directory persists across commands (cd handled specially)
3. Environment variables persist across commands
4. Command history is preserved

### Timeout Behavior

- Default: 30 seconds
- Minimum: 1 second
- Maximum: 3600 seconds (1 hour)
- Adaptive: Some command types auto-extend (apt, docker, etc.)

### SSH Connection Pooling

- Connections persist across multiple commands
- Connection timeout: 10 minutes idle time
- Automatic cleanup of idle connections
- State saved to disk for recovery

### Metadata Availability

Metadata is generated based on request and availability:

- Always available: `command`, `exit_code`, `success`
- Conditional: `registry`, `decision`, `model_hints` (on request)
- Optional in response: `metadata`, `model_hints`, `decision`


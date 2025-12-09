# CLI Tool - Quick Start & Troubleshooting Guide

## Quick Start

### Prerequisites

- Python 3.11+
- pip package manager
- SSH enabled (for remote execution)
- Docker (optional, for containerized deployment)

### Installation

**Step 1: Clone or Setup**
```bash
cd /home/ashish/cli_tool
```

**Step 2: Install Dependencies**
```bash
pip install -r requirements.txt
```

**Step 3: Create Environment File**
```bash
cat > .env << EOF
ENV=development
FASTAPI_PORT=7304
API_SERVER_URL=http://localhost:7304
API_SERVER_DESCRIPTION=Local Command Executor API
EOF
```

**Step 4: Start the Server**

Without Docker:
```bash
python -m uvicorn api:app --host 0.0.0.0 --port 7304 --reload
```

With Docker:
```bash
docker-compose up --build
```

**Step 5: Verify Installation**
```bash
# Check health
curl http://localhost:7304/health

# Expected response
{"status":"healthy","version":"1.0.0"}
```

### First Command

```bash
# Simple local command
curl -X POST http://localhost:7304/api/run-command \
  -H "Content-Type: application/json" \
  -d '{"command": "pwd"}'

# Expected response
{
  "success": true,
  "exit_code": 0,
  "stdout": "/home/ashish/cli_tool",
  "stderr": "",
  "command": "pwd",
  "execution_time": 0.023
}
```

---

## Basic Usage Examples

### Example 1: List Files

```bash
curl -X POST http://localhost:7304/api/run-command \
  -H "Content-Type: application/json" \
  -d '{
    "command": "ls -la /tmp",
    "timeout": 30
  }'
```

### Example 2: Execute with Session

```bash
# Create session
SESSION=$(curl -X POST http://localhost:7304/api/create-session \
  -H "Content-Type: application/json" \
  -d '{"client_info":{"tool":"example"}}' | jq -r '.session_id')

# Execute in session
curl -X POST http://localhost:7304/api/run-command \
  -H "Content-Type: application/json" \
  -d "{
    \"command\": \"pwd\",
    \"session_id\": \"$SESSION\"
  }"

# Get session history
curl http://localhost:7304/api/session/$SESSION/history
```

### Example 3: Remote Execution

```bash
curl -X POST http://localhost:7304/api/local-command-executor \
  -H "Content-Type: application/json" \
  -d '{
    "command": "whoami",
    "remote": true,
    "host": "server.example.com",
    "username": "deploy",
    "password": "secret"
  }'
```

---

## Common Issues & Solutions

### Issue 1: Port Already in Use

**Error:**
```
OSError: [Errno 98] Address already in use
```

**Solution:**
```bash
# Find process using port
lsof -i :7304

# Kill process
kill -9 <PID>

# Or use different port
python -m uvicorn api:app --port 7305
```

---

### Issue 2: Permission Denied for Local Execution

**Error:**
```json
{
  "success": false,
  "exit_code": 126,
  "stderr": "Permission denied"
}
```

**Solution:**
```bash
# Check file permissions
ls -la /path/to/file

# Make executable if needed
chmod +x /path/to/script

# Or use sudo (with appropriate security)
# But note: sudo requires password handling
```

---

### Issue 3: SSH Connection Refused

**Error:**
```json
{
  "error": "Connection refused",
  "detail": "Could not connect to server.example.com"
}
```

**Solution:**
```bash
# Verify SSH is running on remote
ssh -v server.example.com

# Check SSH port
ssh -p 2222 server.example.com  # if not using default 22

# Check credentials
ssh -u deploy server.example.com  # test with username

# Check firewall
ping server.example.com
nc -zv server.example.com 22
```

---

### Issue 4: Command Timeout

**Error:**
```json
{
  "error": "Command execution timeout",
  "detail": "Command exceeded 30 seconds"
}
```

**Solution:**
```bash
# Increase timeout
curl -X POST http://localhost:7304/api/run-command \
  -H "Content-Type: application/json" \
  -d '{
    "command": "long-running-command",
    "timeout": 300  # 5 minutes
  }'

# Note: Maximum timeout is 3600 seconds (1 hour)
```

---

### Issue 5: Docker Build Fails

**Error:**
```
ERROR: failed to solve: ...
```

**Solution:**
```bash
# Clear Docker cache
docker-compose down
docker system prune -a

# Rebuild
docker-compose up --build

# Or rebuild specific service
docker-compose build --no-cache cli_tool
```

---

### Issue 6: SSH Connection Pool Exhaustion

**Error:**
```
Too many open files
```

**Solution:**
```bash
# List active SSH sessions
curl http://localhost:7304/api/ssh-sessions

# Close specific session
curl -X DELETE http://localhost:7304/api/ssh-sessions/session-key

# Close all sessions
# (Iterate through list and delete each)

# Increase file descriptor limit
ulimit -n 4096
```

---

### Issue 7: Command Returns Wrong Output

**Common Cause:** Output encoding issues

**Solution:**
```bash
# Specify encoding
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8

# Restart API
```

---

### Issue 8: High Latency/Slowness

**Diagnosis:**
```bash
# Check execution time in response
curl -X POST http://localhost:7304/api/run-command \
  -H "Content-Type: application/json" \
  -d '{"command": "time ls"}'

# Check system resources
top
df -h
free -h
```

**Solutions:**
1. **Disable metadata generation** if not needed:
   ```bash
   # Don't request metadata
   curl ... -d '{"command": "..."}'  # No metadata field
   ```

2. **Use simpler commands:**
   ```bash
   # Instead of complex pipes
   ls -la | grep txt | wc -l
   
   # Use direct grep
   grep "*.txt" directory
   ```

3. **Increase timeout for slow operations:**
   ```bash
   # Package updates can be slow
   "timeout": 300
   ```

---

### Issue 9: Session Not Persisting State

**Symptom:** Working directory changes or environment variables don't persist.

**Causes & Solutions:**

1. **Not using session:**
   ```bash
   # Wrong: Each request is independent
   curl ... -d '{"command": "cd /tmp"}'
   curl ... -d '{"command": "pwd"}'  # Still in original directory
   
   # Correct: Create session first
   curl ... -d '{"command": "cd /tmp", "session_id": "abc-123"}'
   curl ... -d '{"command": "pwd", "session_id": "abc-123"}'  # Now in /tmp
   ```

2. **Session expired:**
   ```bash
   # Check session status
   curl http://localhost:7304/api/session/abc-123
   
   # Check if expired
   # Response should include last_activity timestamp
   ```

3. **Using different session IDs:**
   ```bash
   # Wrong: Using different session IDs
   curl ... -d '{"command": "cd /tmp", "session_id": "session-1"}'
   curl ... -d '{"command": "pwd", "session_id": "session-2"}'
   
   # Correct: Use same session ID
   curl ... -d '{"command": "cd /tmp", "session_id": "session-1"}'
   curl ... -d '{"command": "pwd", "session_id": "session-1"}'
   ```

---

### Issue 10: Remote Command Requires sudo Password

**Symptom:** Command fails with "sudo: a password is required"

**Solution:**

Option 1: Provide password in request
```bash
curl -X POST http://localhost:7304/api/local-command-executor \
  -H "Content-Type: application/json" \
  -d '{
    "command": "sudo apt-get update",
    "remote": true,
    "host": "server.example.com",
    "username": "deploy",
    "password": "user_password",  # SSH password
    "timeout": 120
  }'
```

Option 2: Configure passwordless sudo (on remote server)
```bash
# On remote server as root
visudo

# Add line (replace 'deploy' with your user)
deploy ALL=(ALL) NOPASSWD: ALL
```

Option 3: Use key-based authentication
```bash
# Create SSH key (on local machine)
ssh-keygen -t rsa -f ~/.ssh/id_rsa

# Copy to remote
ssh-copy-id -i ~/.ssh/id_rsa deploy@server.example.com

# Then configure passwordless sudo on remote
```

---

## Performance Tuning

### 1. Optimize for Large Output

```bash
# Bad: Requesting full metadata for large output
curl ... -d '{"command": "cat large-file.txt", "metadata_format": "full"}'

# Better: Minimal metadata
curl ... -d '{"command": "head -100 large-file.txt", "metadata_format": "minimal"}'

# Or skip metadata entirely
curl ... -d '{"command": "cat large-file.txt"}'
```

### 2. Connection Pooling

SSH connections are automatically pooled. To maximize performance:

```bash
# Good: Use persistent sessions
curl ... -d '{"command": "...", "remote": true, "session_id": "persistent"}'

# Results in connection reuse and state preservation
```

### 3. Command Batching

```bash
# Instead of multiple requests
for cmd in "pwd" "ls" "whoami"; do
  curl ... -d "{\"command\": \"$cmd\"}"
done

# Use single session with multiple commands
SESSION=$(curl ... -d '{}' | jq -r '.session_id')
curl ... -d "{\"command\": \"pwd\", \"session_id\": \"$SESSION\"}"
curl ... -d "{\"command\": \"ls\", \"session_id\": \"$SESSION\"}"
curl ... -d "{\"command\": \"whoami\", \"session_id\": \"$SESSION\"}"
```

### 4. Metadata Format Selection

```bash
# Full metadata: Most detailed, largest payload
"metadata_format": "full"

# Summary metadata: Balanced, recommended for most cases
"metadata_format": "summary"

# Minimal metadata: Only registry info
"metadata_format": "minimal"

# No metadata request: Fastest
# (Don't include metadata_format field)
```

---

## Logging & Debugging

### Enable Detailed Logging

**In environment:**
```bash
export LOG_LEVEL=DEBUG
python -m uvicorn api:app --reload --log-level debug
```

**In code:**
```python
import logging
logging.basicConfig(level=logging.DEBUG)
```

### View Logs

```bash
# API logs
tail -f logs/api_system.log

# Command logs
tail -f logs/api_cmd.log

# Metadata logs
tail -f logs/api_metadata.log

# Decision logs
tail -f logs/api_decision.log
```

### Debug SSH Connection

```bash
# Use verbose SSH
ssh -vvv deploy@server.example.com

# Check paramiko logs in Python
import logging
logging.getLogger("paramiko").setLevel(logging.DEBUG)
```

### Test Command Before API

```bash
# Test locally first
ls -la /tmp

# Then via API
curl -X POST http://localhost:7304/api/run-command \
  -H "Content-Type: application/json" \
  -d '{"command": "ls -la /tmp"}'
```

---

## Production Checklist

- [ ] Enable authentication (configure JWT)
- [ ] Use HTTPS only
- [ ] Restrict CORS origins
- [ ] Enable rate limiting
- [ ] Configure firewall rules
- [ ] Setup log rotation and archival
- [ ] Configure automated backups for session history
- [ ] Setup monitoring and alerting
- [ ] Test disaster recovery procedures
- [ ] Document deployment architecture
- [ ] Review security policies
- [ ] Setup API key rotation
- [ ] Enable audit logging
- [ ] Configure SSH key rotation
- [ ] Test performance under load

---

## Useful Commands

### Health Check
```bash
curl http://localhost:7304/health
```

### List Sessions
```bash
curl http://localhost:7304/api/ssh-sessions
```

### Create Session
```bash
curl -X POST http://localhost:7304/api/create-session \
  -H "Content-Type: application/json" \
  -d '{}'
```

### Execute Command
```bash
curl -X POST http://localhost:7304/api/run-command \
  -H "Content-Type: application/json" \
  -d '{"command": "pwd"}'
```

### Get Session History
```bash
curl http://localhost:7304/api/session/{SESSION_ID}/history
```

### Stop API Server
```bash
# Graceful shutdown
Ctrl+C

# Force stop (if hung)
pkill -9 -f "uvicorn api:app"
```

---

## Further Reading

- [Architecture Documentation](ARCHITECTURE.md)
- [API Reference](API_REFERENCE.md)
- [Module Reference](MODULES_REFERENCE.md)
- [Security Documentation](SECURITY.md) (if available)

---

## Getting Help

### Check Logs
```bash
tail -f logs/*.log
```

### Test Connectivity
```bash
# API
curl http://localhost:7304/health

# SSH (if using remote)
ssh -v user@host

# Specific port
nc -zv server 22
```

### Review Configuration
```bash
# Check environment
env | grep -i api

# Check .env file
cat .env
```

### Restart Services
```bash
# Without Docker
pkill -f "uvicorn api:app"
python -m uvicorn api:app --host 0.0.0.0 --port 7304

# With Docker
docker-compose restart
```


# Dashboard Runner User Guide

Complete guide for using the AgentPing Dashboard Runner to manage and monitor your dashboard applications.

Status note:

- Canonical runtime startup/verification commands are in `community/agentping/docs/getting-started.md`.
- Use this file as the deep operational companion for runner behavior and troubleshooting.

---

## Table of Contents

1. [Overview](#overview)
2. [Getting Started](#getting-started)
3. [Adding a Dashboard](#adding-a-dashboard)
4. [Managing Dashboards](#managing-dashboards)
5. [Finding Logs](#finding-logs)
6. [Troubleshooting](#troubleshooting)

---

## Overview

The Dashboard Runner provides resilient process management for your dashboard applications with:

- **Auto-restart** - Automatically recovers from crashes with intelligent backoff
- **Smart port selection** - Resolves port conflicts automatically
- **Health monitoring** - Continuous health checks with configurable intervals
- **Real-time status** - Live dashboard status updates in the Navigator UI
- **Comprehensive logging** - Structured logs with automatic rotation

---

## Getting Started

### Prerequisites

The Dashboard Runner is integrated into AgentPing Studio. Ensure you have:

1. AgentPing Studio installed
2. Dashboard applications configured in `community/agentping/packages/dashboard-runner/config/dashboards.yaml`
3. Node.js or pnpm installed (for running dashboards)

### Starting Dashboard Runner + Manager UI

Launch from the AgentPing workspace:

```bash
cd ~/digital/leviathan/community/agentping
pnpm dashboards:start
```

Open the manager UI:

- `http://localhost:5175` (Dashboard Manager UI)
- `http://localhost:3030/api/dashboards` (Runner API)

If you also want Studio shell, run in another terminal:

```bash
cd ~/digital/leviathan/community/agentping
pnpm --filter @agentping/studio dev
```

---

## Adding a Dashboard

### Step 1: Edit Configuration File

Open the dashboard configuration file:

```bash
~/digital/leviathan/community/agentping/packages/dashboard-runner/config/dashboards.yaml
```

### Step 2: Add Dashboard Entry

Add a new dashboard entry to the `dashboards` array:

```yaml
dashboards:
  - name: My Dashboard              # Display name shown in UI
    id: my-dashboard                # Unique identifier (lowercase, hyphens)
    port: 3000                      # Preferred port number
    port_range: [3000, 3004]        # Fallback ports if preferred is occupied
    command: npm run dev -- --port {port}  # Command to start dashboard
    cwd: /path/to/dashboard         # Working directory (supports ~ for home)
    health_check:
      type: http                    # Check type: http or process
      path: /                       # HTTP endpoint to check
      timeout_ms: 5000              # Request timeout
      expected_status: 200          # Expected HTTP status code(s)
      interval_ms: 10000            # Check interval (10 seconds)
    restart_policy:
      enabled: true                 # Enable auto-restart
      max_retries: 5                # Maximum restart attempts
      backoff_ms: [1000, 2000, 4000, 8000, 16000]  # Exponential backoff delays
```

### Step 3: Configuration Options

#### Required Fields

| Field | Description | Example |
|-------|-------------|---------|
| `name` | Display name in Navigator UI | `"Sofia UI Storybook"` |
| `id` | Unique identifier (used for logs, metrics) | `"sofia"` |
| `port` | Preferred port number | `6007` |
| `command` | Shell command to start the dashboard | `"pnpm storybook --port {port}"` |
| `cwd` | Working directory for the command | `"~/digital/leviathan/community/agentping/packages/ui"` |

#### Optional Fields

| Field | Default | Description |
|-------|---------|-------------|
| `port_range` | `[]` | Array of fallback ports if preferred is occupied |
| `health_check.type` | `"http"` | Health check type: `http` or `process` |
| `health_check.path` | `"/"` | HTTP endpoint to check |
| `health_check.timeout_ms` | `5000` | Request timeout in milliseconds |
| `health_check.expected_status` | `200` | Expected HTTP status code(s) - single number or array |
| `health_check.interval_ms` | `10000` | Health check interval (10 seconds) |
| `restart_policy.enabled` | `true` | Enable auto-restart on crash |
| `restart_policy.max_retries` | `5` | Maximum restart attempts before giving up |
| `restart_policy.backoff_ms` | `[1000, 2000, 4000, 8000, 16000]` | Exponential backoff delays |

#### Port Placeholder

Use `{port}` in your command to inject the selected port:

```yaml
command: pnpm dev --port {port}
command: npm start -- -p {port}
command: bunx vite --port {port}
```

The runner will replace `{port}` with the actual port selected (from preferred or port_range).

### Step 4: Restart AgentPing Studio

After adding or modifying dashboards:

1. Close AgentPing Studio
2. Restart with `pnpm dashboards:start`
3. Your new dashboard will appear in the Navigator

---

## Managing Dashboards

### Dashboard Navigator UI

The Navigator displays all configured dashboards with real-time status:

#### Status Indicators

| Status | Icon | Description |
|--------|------|-------------|
| **ONLINE** | 🟢 Green checkmark | Dashboard running and healthy |
| **OFFLINE** | 🔴 Red X | Dashboard not responding to health checks |
| **RESTARTING** | 🔄 Spinning arrows | Auto-restart in progress |
| **FAILED** | ❌ Red X | Max restart attempts exceeded |
| **CHECKING** | ⚠️ Yellow alert | Initial health check in progress |

### Restarting a Dashboard from UI

#### Manual Restart

For dashboards with status **OFFLINE** or **FAILED**:

1. Locate the dashboard card in Navigator
2. Click the **Restart** button at the bottom of the card
3. Status will change to **RESTARTING**
4. Wait for auto-restart completion (typically 1-5 seconds)
5. Status will update to **ONLINE** if successful

#### Automatic Restart

When a dashboard crashes:

1. Runner detects the crash immediately
2. Status changes to **RESTARTING**
3. Runner waits according to backoff schedule:
   - 1st attempt: 1 second delay
   - 2nd attempt: 2 seconds delay
   - 3rd attempt: 4 seconds delay
   - 4th attempt: 8 seconds delay
   - 5th attempt: 16 seconds delay
4. After max retries (default: 5), status changes to **FAILED**

#### Opening a Dashboard

For **ONLINE** dashboards:

1. Click anywhere on the dashboard card
2. Dashboard opens in your default browser
3. Use the displayed URL or port number

### Port Auto-Selection

If a dashboard's preferred port is occupied:

1. Runner tries each port in `port_range`
2. If all ports occupied, finds any available port in [6000-9000]
3. Navigator displays: `PORT 3002 (auto-selected)`
4. Dashboard URL updates automatically

---

## Finding Logs

### Log Directory Structure

All logs are stored in:

```
~/.local/share/lev/dashboard-runner/logs/
```

### Log Files

| File | Description |
|------|-------------|
| `runner.log` | Main runner log (startup, shutdown, system events) |
| `{dashboard-id}.log` | Per-dashboard logs (e.g., `agentping.log`, `sofia.log`) |

### Viewing Logs

#### Real-Time Log Streaming (Recommended)

```bash
# Stream main runner log
tail -f ~/.local/share/lev/dashboard-runner/logs/runner.log

# Stream specific dashboard log
tail -f ~/.local/share/lev/dashboard-runner/logs/agentping.log
tail -f ~/.local/share/lev/dashboard-runner/logs/sofia.log
```

#### View Last 100 Lines

```bash
tail -n 100 ~/.local/share/lev/dashboard-runner/logs/agentping.log
```

#### Search Logs for Errors

```bash
grep -i error ~/.local/share/lev/dashboard-runner/logs/agentping.log
grep -i crash ~/.local/share/lev/dashboard-runner/logs/runner.log
```

### Log Format

Logs use structured JSON format:

```json
{
  "timestamp": "2026-01-31T19:30:45.123Z",
  "level": "info",
  "dashboardId": "agentping",
  "message": "Process started",
  "port": 6006,
  "pid": 12345
}
```

### Log Levels

| Level | Description |
|-------|-------------|
| `info` | Normal operations (startup, shutdown, health checks) |
| `warn` | Warnings (health check failures, port conflicts) |
| `error` | Errors (crashes, restart failures) |
| `debug` | Detailed debugging information |

### Log Rotation

Logs automatically rotate when they reach **10MB**:

- Old log: `agentping.log.1`
- Current log: `agentping.log`

---

## Troubleshooting

### Dashboard Shows OFFLINE

**Possible Causes:**
1. Dashboard process crashed
2. Port conflict preventing startup
3. Health check endpoint incorrect
4. Dashboard taking too long to start

**Solutions:**

1. **Check logs for errors:**
   ```bash
   tail -f ~/.local/share/lev/dashboard-runner/logs/{dashboard-id}.log
   ```

2. **Verify port availability:**
   ```bash
   lsof -i :{port}
   # Kill process if needed
   kill {PID}
   ```

3. **Check health check path:**
   - Verify `health_check.path` matches actual dashboard route
   - Example: If dashboard serves `/app`, set `path: /app`

4. **Increase timeout:**
   ```yaml
   health_check:
     timeout_ms: 10000  # Increase from 5000 to 10000
   ```

5. **Click Restart button** in Navigator UI

---

### Dashboard Shows FAILED

**Possible Causes:**
1. Max restart attempts exceeded (5 by default)
2. Dashboard configuration error
3. Missing dependencies or files
4. Permission issues

**Solutions:**

1. **Check logs for crash reason:**
   ```bash
   grep -A 5 "crashed" ~/.local/share/lev/dashboard-runner/logs/{dashboard-id}.log
   ```

2. **Verify dashboard can run manually:**
   ```bash
   cd {dashboard-cwd}
   {dashboard-command}
   # Example: pnpm storybook --port 6006
   ```

3. **Check file permissions:**
   ```bash
   ls -la {dashboard-cwd}
   ```

4. **Fix configuration errors:**
   - Check `dashboards.yaml` syntax
   - Verify `cwd` path exists
   - Ensure `command` is correct

5. **Increase max retries:**
   ```yaml
   restart_policy:
     max_retries: 10  # Increase from 5
   ```

6. **Click Restart button** after fixing the issue

---

### Dashboard Shows RESTARTING (Stuck)

**Possible Causes:**
1. Dashboard startup time exceeds health check timeout
2. Zombie processes preventing port binding
3. Infinite restart loop

**Solutions:**

1. **Check if process is actually running:**
   ```bash
   ps aux | grep {dashboard-command}
   ```

2. **Kill zombie processes:**
   ```bash
   lsof -i :{port}
   kill -9 {PID}
   ```

3. **Increase health check timeout:**
   ```yaml
   health_check:
     timeout_ms: 10000
     interval_ms: 15000
   ```

4. **Restart AgentPing Studio:**
   - Close AgentPing Studio
   - Kill any lingering processes
   - Restart with `pnpm start`

---

### Port Auto-Selected (Unexpected)

**Possible Causes:**
1. Another process using preferred port
2. Previous dashboard instance still running

**Solutions:**

1. **Find process using the port:**
   ```bash
   lsof -i :{preferred-port}
   ```

2. **Kill the process:**
   ```bash
   kill {PID}
   ```

3. **Restart dashboard** from Navigator UI

4. **Or expand port_range:**
   ```yaml
   port_range: [3000, 3010]  # More fallback options
   ```

---

### Auto-Restart Not Working

**Possible Causes:**
1. Auto-restart disabled in configuration
2. Dashboard runner not initialized
3. IPC communication failure

**Solutions:**

1. **Check restart policy:**
   ```yaml
   restart_policy:
     enabled: true  # Must be true
   ```

2. **Verify runner badge in Navigator:**
   - Look for "✨ Auto-Restart Enabled" badge
   - If missing, runner not initialized

3. **Check main process logs:**
   ```bash
   tail -f ~/.local/share/lev/dashboard-runner/logs/runner.log
   ```

4. **Restart AgentPing Studio** to reinitialize runner

---

### Health Checks Failing for Working Dashboard

**Possible Causes:**
1. Health check path incorrect
2. Expected status code mismatch
3. CORS or network issues
4. Dashboard redirecting requests

**Solutions:**

1. **Verify health check endpoint:**
   ```bash
   curl -I http://localhost:{port}{path}
   # Example: curl -I http://localhost:6006/
   ```

2. **Check actual status code:**
   - If dashboard returns 307 (redirect), update config:
   ```yaml
   health_check:
     expected_status: [200, 307]  # Allow both
   ```

3. **Change to process-only check:**
   ```yaml
   health_check:
     type: process  # Only check if PID exists
   ```

4. **Adjust path:**
   ```yaml
   health_check:
     path: /health  # Custom health endpoint
   ```

---

### Logs Not Appearing

**Possible Causes:**
1. Log directory doesn't exist
2. Permission issues
3. Dashboard not outputting logs

**Solutions:**

1. **Create log directory:**
   ```bash
   mkdir -p ~/.local/share/lev/dashboard-runner/logs
   ```

2. **Check permissions:**
   ```bash
   ls -la ~/.local/share/lev/dashboard-runner/logs
   chmod 755 ~/.local/share/lev/dashboard-runner/logs
   ```

3. **Verify dashboard process is running:**
   ```bash
   ps aux | grep {dashboard-id}
   ```

4. **Force log output:**
   - Ensure dashboard outputs to stdout/stderr
   - Runner captures both streams automatically

---

### Multiple Instances Running

**Possible Causes:**
1. AgentPing Studio started multiple times
2. Manual dashboard startup + runner startup
3. Zombie processes

**Solutions:**

1. **Kill all dashboard processes:**
   ```bash
   pkill -f "pnpm storybook"
   pkill -f "pnpm dev"
   # Etc. for your specific commands
   ```

2. **Find all instances:**
   ```bash
   ps aux | grep storybook
   ps aux | grep "dashboard-runner"
   ```

3. **Kill by port:**
   ```bash
   lsof -ti :{port} | xargs kill -9
   ```

4. **Restart AgentPing Studio cleanly**

---

## Advanced Tips

### Custom Health Check Scripts

For complex health checks, create a custom endpoint in your dashboard:

```typescript
// dashboard/health.ts
app.get('/health', (req, res) => {
  const checks = {
    database: checkDatabase(),
    cache: checkCache(),
    services: checkServices()
  };

  const healthy = Object.values(checks).every(c => c === true);
  res.status(healthy ? 200 : 503).json(checks);
});
```

Then configure:

```yaml
health_check:
  path: /health
  expected_status: 200
```

### Disable Auto-Restart for Development

When actively developing and expecting crashes:

```yaml
restart_policy:
  enabled: false
```

### Custom Backoff Strategy

For critical dashboards that need faster recovery:

```yaml
restart_policy:
  backoff_ms: [500, 1000, 2000]  # Faster, fewer retries
```

For non-critical dashboards:

```yaml
restart_policy:
  backoff_ms: [5000, 10000, 20000, 40000]  # Slower, more patient
```

### Environment Variables

Pass environment variables in the command:

```yaml
command: PORT={port} NODE_ENV=production npm start
```

### Multiple Port Ranges

For dashboards that commonly conflict:

```yaml
port_range: [6006, 6007, 6008, 6009, 6010, 6020, 6021, 6022]
```

---

## Need Help?

1. **Check logs first:** `~/.local/share/lev/dashboard-runner/logs/`
2. **Review configuration:** `packages/dashboard-runner/config/dashboards.yaml`
3. **Test dashboard manually:** Run command from `cwd` to verify it works
4. **Restart AgentPing Studio:** Often resolves transient issues
5. **Report issues:** If problem persists, check documentation or file a bug report

---

**Last Updated:** 2026-01-31
**Version:** 1.0.0

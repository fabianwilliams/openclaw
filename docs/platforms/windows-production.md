---
summary: "Windows production hardening: keep-alive, startup ordering, permissions, and known issues"
title: Windows Production Hardening
read_when:
  - Running OpenClaw in production on Windows (WSL2 or native)
  - Debugging Gateway reliability issues on Windows
  - Setting up unattended Windows deployment
---

# Windows Production Hardening

This guide covers hardening an OpenClaw Gateway running on Windows for
production reliability. It assumes you have completed the basic
[Windows (WSL2)](/platforms/windows) setup.

Production means: the Gateway runs unattended, recovers from crashes, and
stays operational across reboots and updates.

## Prerequisites

- OpenClaw installed and working per [Windows (WSL2)](/platforms/windows).
- Gateway daemon installed via `openclaw gateway install` or `openclaw onboard --install-daemon`.
- WSL2 with systemd enabled.

## Startup orchestration

When running multiple services (Signal CLI, Gateway, companion processes),
**startup order matters**. The Gateway expects channel backends (like Signal CLI)
to be reachable at boot. If Signal CLI is not running when the Gateway starts,
the Signal channel fails silently until restart.

### Problem: race condition at boot

Both the Gateway and Signal CLI start as systemd services. If the Gateway
initializes before Signal CLI is listening, Signal messages are dropped.

### Solution: systemd ordering

Add a dependency so the Gateway waits for Signal CLI:

```ini
# ~/.config/systemd/user/openclaw-gateway.service.d/override.conf
[Unit]
After=signal-cli-rest-api.service
Wants=signal-cli-rest-api.service
```

Create the override:

```bash
mkdir -p ~/.config/systemd/user/openclaw-gateway.service.d
cat > ~/.config/systemd/user/openclaw-gateway.service.d/override.conf << 'EOF'
[Unit]
After=signal-cli-rest-api.service
Wants=signal-cli-rest-api.service
EOF

systemctl --user daemon-reload
systemctl --user restart openclaw-gateway
```

For other channel backends, replace `signal-cli-rest-api.service` with the
appropriate service name.

## Keep-alive watchdog

The Gateway process can exit unexpectedly — Node.js crashes, auto-updates,
OOM kills. A watchdog restarts it automatically.

### Option A: systemd (recommended for WSL2)

If you used `openclaw gateway install`, systemd already manages restarts via
`Restart=on-failure`. Verify:

```bash
systemctl --user show openclaw-gateway | grep Restart=
# Should show: Restart=on-failure
```

Check status:

```bash
systemctl --user status openclaw-gateway --no-pager
journalctl --user -u openclaw-gateway --since "1 hour ago" --no-pager | tail -20
```

### Option B: Windows Scheduled Task (native Windows)

For deployments running OpenClaw directly on Windows (not WSL2), use a
Scheduled Task as a watchdog:

```powershell
# Create a health-check script
$ScriptPath = "C:\Tools\openclaw\check-gateway.ps1"

@'
$proc = Get-Process -Name "node" -ErrorAction SilentlyContinue |
  Where-Object { $_.CommandLine -like "*openclaw*" }

if (-not $proc) {
    Start-Process -FilePath "C:\Tools\openclaw\start-openclaw.ps1" `
      -WindowStyle Hidden
    Add-Content "C:\Tools\openclaw\watchdog.log" `
      "$(Get-Date -Format o) Gateway restarted by watchdog"
}
'@ | Set-Content $ScriptPath

# Register as Scheduled Task (runs every 5 minutes)
schtasks /create /tn "OpenClaw-Gateway-KeepAlive" `
  /tr "powershell.exe -ExecutionPolicy Bypass -File $ScriptPath" `
  /sc minute /mo 5 /ru "$env:USERNAME"
```

> **Important**: The watchdog restarts the _process_ but cannot fix expired
> authentication tokens. If the Gateway is running but channels fail, check
> the auth state (see [Known issues](#known-issues)).

## File permissions

Secure the OpenClaw state directory to prevent unauthorized access to
credentials and session data.

### WSL2 (Linux permissions)

```bash
# Lock down the state directory
chmod 700 ~/.openclaw
chmod 600 ~/.openclaw/openclaw.json

# Secure auth profiles (contain API keys and tokens)
find ~/.openclaw -name "auth-profiles.json" -exec chmod 600 {} \;

# Secure cron store
chmod 600 ~/.openclaw/cron/jobs.json 2>/dev/null
```

### Native Windows (NTFS ACLs)

```powershell
$OpenClawDir = "$env:USERPROFILE\.openclaw"

# Remove inherited permissions and grant only current user
icacls $OpenClawDir /inheritance:r /grant "${env:USERNAME}:(OI)(CI)F" /T
```

## Security audit

Run the built-in security audit to catch misconfigurations:

```bash
openclaw security audit --deep --fix
```

Review the output for:

- Open file permissions on credential files.
- Overly broad tool policies.
- Missing sandbox configuration for multi-agent setups.
- Stale auth profiles from removed agents.

### Windows-specific audit items

Manual checks beyond what `security audit` covers:

1. **Firewall rules**: verify only required ports are open.

```powershell
Get-NetFirewallRule -DisplayName "*WSL*" | Format-Table DisplayName, Direction, Action
```

2. **Scheduled Tasks**: review all OpenClaw-related tasks.

```powershell
schtasks /query /tn "OpenClaw*" /fo LIST /v
```

3. **WSL networking**: confirm the Gateway is not exposed to the public internet.

```bash
# Inside WSL — should show 127.0.0.1 or WSL internal IP only
ss -tlnp | grep node
```

## Monitoring

### Health check endpoint

If the Gateway exposes a health endpoint, poll it from the watchdog:

```bash
# Quick health check from WSL
curl -sf http://127.0.0.1:3000/health || echo "Gateway unhealthy"
```

### Log monitoring

Watch for error patterns that indicate degraded operation:

```bash
# Recent errors
journalctl --user -u openclaw-gateway --since "1 hour ago" --priority err --no-pager

# Auth failures (often the first sign of token expiry)
journalctl --user -u openclaw-gateway --since "1 hour ago" --no-pager | grep -i "auth\|token\|refresh\|unauthorized"
```

### Cron job health

Verify cron jobs are running on schedule:

```bash
openclaw cron list
openclaw cron runs --id <job-id> --limit 5
```

Look for jobs with consecutive failures — they may indicate auth issues or
network problems rather than prompt errors.

## Known issues

### OAuth token refresh failures

**Symptom**: Gateway is running, watchdog shows healthy, but channels stop
responding. Logs show `refresh_token_reused` or similar auth errors.

**Cause**: Some OAuth providers (notably OpenAI for Codex-based models)
invalidate refresh tokens on reuse. A Gateway restart or auto-update can
trigger a double-use, permanently breaking the token.

**Fix**: Re-authenticate manually.

```bash
openclaw configure --section model
# Select the OAuth provider and complete browser sign-in
```

For unattended deployments, prefer API key authentication over OAuth where
the provider supports it. API keys do not expire on process restart.

### Azure AD usernames with special characters

**Symptom**: Signal CLI or other channel backends fail to authenticate when
the Azure AD username contains parentheses or other special characters.

**Workaround**: Create an alias or secondary user principal name without
special characters. Use the alias in the OpenClaw channel configuration.

### Auto-update process crashes

**Symptom**: Gateway exits during an auto-update and does not restart cleanly.

**Fix**: The watchdog (systemd or Scheduled Task) handles this automatically.
If the updated version fails to start, pin a known-good version:

```bash
# Pin to a specific version
npm install -g openclaw@2026.1.6
```

### WSL memory pressure

**Symptom**: Gateway or Node.js killed by OOM on WSL2.

**Fix**: Set a memory limit in `.wslconfig` (Windows side):

```ini
# %USERPROFILE%\.wslconfig
[wsl2]
memory=8GB
swap=4GB
```

Restart WSL after editing:

```powershell
wsl --shutdown
```

## Production checklist

Before considering a Windows deployment production-ready:

- [ ] Gateway daemon installed (`systemctl --user is-enabled openclaw-gateway`).
- [ ] Startup ordering configured (channel backends before Gateway).
- [ ] Keep-alive watchdog active (systemd or Scheduled Task).
- [ ] File permissions locked down (`chmod 700 ~/.openclaw`).
- [ ] Security audit passing (`openclaw security audit --deep`).
- [ ] Cron jobs verified (`openclaw cron list` + manual `cron run` test).
- [ ] Auth method chosen (prefer API keys over OAuth for unattended).
- [ ] Log monitoring in place (error grep or health endpoint).
- [ ] WSL memory limit set (`.wslconfig`).
- [ ] Backup strategy for `~/.openclaw/` (credentials, cron store, sessions).

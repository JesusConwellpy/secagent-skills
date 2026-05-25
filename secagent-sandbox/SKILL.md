---
name: secagent-sandbox
description: Secure execution protocol. Pre-execution risk assessment, authorization check, execution audit logging.
tools: exec_shell, write_file, read_file, fetch_url
---

# SecAgent Sandbox

Loading this SKILL enforces a security protocol before any tool execution.

## Authorization Check (First step — mandatory)

```
1. Does the target match *.ctfhub.com, *.hackthebox.com, *.tryhackme.com?
   → YES: CTF-exempt. Proceed.

2. Is the target localhost/127.0.0.1?
   → YES: Proceed.

3. Check ~/.secagent/authorization.toml:
   - CIDR match? → Proceed.
   - Domain wildcard match? → Proceed.
   - File doesn't exist? → Create it (empty file = deny-all except localhost + CTF).

4. None of the above?
   → REJECT. Output: "Target {target} not authorized."
```

## authorization.toml Bootstrap

```
If ~/.secagent/authorization.toml doesn't exist:

mkdir -p ~/.secagent
touch ~/.secagent/authorization.toml

Default content:
# SecAgent Authorization
# Add authorized targets below.
# CIDR example: 192.168.0.0/16
# Domain example: *.example.com
# CTF platforms (*.ctfhub.com, *.hackthebox.com, *.tryhackme.com) are auto-authorized.

Default behavior: empty file → allow only localhost + CTF platforms.
```

## Operation Risk Levels

```
Read file (cat, head, read)      → LOW (auto-approved)
curl GET (read network)          → LOW (auto-approved, authorized targets only)
Write file (write, edit)         → MEDIUM (auto-approved inside workspace)
nmap (network scan)              → MEDIUM (authorized targets only)
Execute PoC (exploit)            → HIGH (allowed, logged)
Dangerous command                → CRITICAL (rejected or reported)

Dangerous patterns (immediate rejection):
  bash -c "..." (inline command execution)
  curl ... | bash (pipe to shell)
  $(...) or `` (command substitution)
  /dev/tcp (reverse shell)
  nc -e / ncat -e (netcat reverse shell)
  >/dev/null 2>&1 with dangerous commands (output suppression)
```

## Audit Log

```
Log one line per operation:

[EXEC] {ISO8601_UTC} | {tool} | exit={code} | {duration_ms}ms | {risk}

  ISO8601_UTC: 2026-05-25T10:30:00Z format, always UTC
  exit_code: 0-255, or BLOCKED, or TIMEOUT
  duration_ms: total time from assessment to completion; assessment time if blocked
  risk: LOW / MEDIUM / HIGH / CRITICAL

Examples:
[EXEC] 2026-05-25T10:30:01Z | curl | exit=0 | 230ms | LOW
[EXEC] 2026-05-25T10:31:00Z | curl | exit=BLOCKED | 5ms | HIGH
```

## CTF-Specific Rules

```
CTF platform targets (*.ctfhub.com, etc.):
  - All read operations: auto-approved (CTF exemption)
  - PoC execution: allowed (target is designed to be attacked)
  - Write operations: normal assessment (no exemption)

CTF environments don't need:
  - Sandbox isolation (target already sandboxed)
  - Defense bypass (target expects attacks)
  - User approval (CTF is self-paced)
```

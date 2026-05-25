---
name: secagent-orchestration
description: Sub-agent orchestration. 9 security agent types, parallel fan-out, mailbox communication, structured result synthesis.
tools: agent_spawn, agent_wait, agent_result, agent_cancel, agent_list
---

# SecAgent Orchestration — Sub-agent Coordination

Loading this SKILL gives you multi-agent coordination capabilities.

## Available Agent Types

```
spawn type parameter:

recon           Recon: port scan, service discovery, subdomains, OSINT
web-analyst     Web: SQLi/XSS/LFI/SSRF/CSRF detection
binary-analyst  Binary: checksec, disassemble, ROP/shellcode
exploit-runner  Exploit: PoC execution, vuln verification (sandboxed)
intel-gatherer  Intel: CVE search, exploit-db, GitHub PoC
code-auditor    Audit: source code vulnerability discovery
post-exploit    Post-exploit: persistence, lateral movement
coordinator     Coordination: team orchestration, evidence synthesis
general         General: non-security-specific tasks
```

## Orchestration Rules

### Fan-Out (most important)

```
N independent tasks = N agent_spawn calls in the SAME turn. Never serialize.

Correct:
  Same turn: spawn recon "target A" + spawn recon "target B" + spawn intel "CVE search"
  → 3 agents run in parallel

Wrong:
  Wait for A → then spawn B → serial waste
```

### Trigger Conditions

```
>1 target                  → spawn recon × N (parallel)
Product + version found    → spawn intel-gatherer (immediately)
Hypothesis confidence ≥0.7  → spawn specialist for verification
Exploitation path clear    → spawn exploit-runner
Binary file provided       → spawn binary-analyst
```

### Communication Protocol

```
After spawn, poll for completion:
  agent_wait(id)  → blocks until agent finishes, returns {id, status, result}
  agent_result(id)→ pulls structured output for a completed agent

On completion:
  1. Read result summary
  2. Integrate — don't redo the sub-agent's work
  3. Call agent_result(id) for full output if the summary is too thin
```

### Verification Protocol

```
Iron rule: ONE hypothesis per agent.

1. spawn exploit-runner with EXACT CVE ID + target
2. Returns: success/failure + evidence
3. Success → CONFIRMED, advance
4. Failure → 3-step analysis:
   a. Hypothesis wrong? → new hypothesis, new agent
   b. Test wrong? → fix, retry
   c. Target limited it? (WAF, patch, firewall) → different path or document exclusion
```

## Forbidden Actions

```
1. Direct security operations → Wrong. Spawn agent.
2. One agent testing multiple hypotheses → Wrong. One per agent.
3. Agent failure with no analysis → Wrong. Must form new hypothesis.
4. Concluding before results arrive → Wrong. Wait for agent.
5. Serial execution of parallelizable work → Wrong. Same-turn fan-out.
```

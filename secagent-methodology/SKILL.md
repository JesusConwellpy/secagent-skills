---
name: secagent-methodology
description: Universal security attack methodology. 4-phase pentest cycle, vulnerability classification, evidence-driven decision-making. Works for CTF, pentest, code audit, binary analysis.
tools: exec_shell, web_search, fetch_url, read_file, write_file, agent_spawn
---

# SecAgent Methodology

Loading this SKILL gives you a universal security workflow. It adapts to any task type.

## Task Classification (First step)

```
Classify before acting:

1. CTF challenge?
   → Solo execution. Analyze challenge directly. No agent spawn.
   → Key: read source → identify all checkpoints → bypass each one

2. Pentest (multi-target/multi-service)?
   → Use agents. Spawn in parallel. Full Phase 1-4.

3. Code audit?
   → Solo. Read file by file → trace data flow → mark vulnerability points.

4. Binary analysis?
   → Solo. checksec → disassemble → find vuln → write exploit.

5. CVE research?
   → Solo. Search → match → verify.
```

## Phase 1: RECON — Know your target

```
1. Gather basic info:
   - Web: curl -s <url> | head -100; curl -s -I <url>
   - Binary: file <binary>; checksec <binary>; strings <binary>
   - Code: read entry file → trace includes → read all related files

2. Identify tech stack:
   - Language, framework, version
   - Entry points (URL params, API endpoints, file inputs)
   - Security mechanisms (WAF, auth, input validation)

3. If CTF:
   - Read complete source
   - List all inputs: GET/POST params, headers, cookies, file uploads
   - List all checkpoints: if/die statements, preg_match, filter_var, escapes
```

## Phase 2: ANALYZE — From observation to hypothesis

```
1. Map data flow:
   Input → Validation/Filter → Processing → Output
   At each step: who controls the data? what constraints exist?

2. Identify vulnerability categories:
   Web:     SQLi, XSS, RCE, LFI, SSRF, CSRF, IDOR, SSTI, File Upload
   Binary:  BOF, Format String, UAF, Integer Overflow, Race Condition
   Crypto:  Weak cipher, Padding Oracle, Hash extension, Nonce reuse
   General: Auth bypass, PrivEsc, Info disclosure, Logic flaw

3. For each checkpoint, ask:
   - What does it check? (type, length, format, range, origin)
   - Bypass method? (type juggling, encoding, null byte, overflow, race)
   - Is this checkpoint unnecessary? (alternative path? later code overwrites it?)

4. Identify red herrings:
   - Computationally infeasible checks (hash preimage, collision)
   - Dead code (after die())
   - Variables overwritten by later steps

5. Form hypotheses: testable, falsifiable. Give confidence 0-1.
```

## Phase 3: EXPLOIT — Verify hypotheses

```
1. Test highest confidence first

2. Build minimal PoC:
   - Web: curl -s "<url>?param=value"
   - Binary: python -c "print 'A'*N + p32(addr)"
   - Change ONE parameter at a time. Observe before adjusting.

3. Observe output:
   - flag{...} / success → CONFIRMED
   - Different error → adjust parameter
   - Same error → different technique
   - Shell tip: wrap curl URLs in single quotes to prevent $ expansion

4. Failure analysis (3-step):
   a. Hypothesis wrong? → new hypothesis
   b. Test wrong? → fix, retry
   c. Target limited it? → different path
```

## Phase 4: REPORT

```
## Flag/Result: {outcome}

## Exploit Chain:
| Step | What | How | Payload |
|------|------|-----|---------|
| 1    | ...  | ... | ...     |

## Key Insight:
{the most important discovery — red herring? key bypass? data flow flaw?}
```

## Quick Reference: Common Vulnerability Patterns

```
Type juggling (weak-type languages: PHP/JS):
  string=="N" ≠ int==N, "Nabc"==N (loose comparison)

Wrappers/protocols:
  file_get_contents: data://, php://, expect://, file://
  SSRF: file://, gopher://, dict://, http://internal

Regex bypass:
  Unicode homoglyphs (U+FF04 ≠ U+0024)
  Multi-byte encoding (UTF-8 overlong, illegal sequences)
  Backtrack limits (PCRE backtrack limit → error)

Serialization/deserialization:
  PHP unserialize, Python pickle, Java deserialization, Node serialize
  Magic methods: __destruct, __wakeup, __toString

Command injection:
  Concat: ; | & && || ` $()
  Space bypass: $IFS, ${IFS}, <, <>
  Keyword bypass: ca\t, c'a't, c"a"t, /???/c?t
```

## Execution Rules

```
- Always understand the target first. Never skip Phase 1.
- Wrap curl URLs in single quotes to prevent shell $ expansion.
- Change one parameter at a time. Observe before next step.
- CTF challenges don't need agent spawn — solve directly.
- Stop when you see the flag. Record the exploit chain.
```

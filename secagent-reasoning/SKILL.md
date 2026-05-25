---
name: secagent-reasoning
description: Detective reasoning engine. Clue extraction, cross-correlation, 12 anomaly detectors, failure pattern matching, attack chain detection.
tools: read_file, write_file, grep_files
---

# SecAgent Reasoning — Detective Engine

Loading this SKILL gives you structured security reasoning.

## Reasoning Workflow

```
Input: tool output / agent report / scan result

Step 1: CLUE EXTRACTION → extract all clues from raw output
Step 2: CROSS-CORRELATION → find relationships between clues
Step 3: ANOMALY DETECTION → run applicable detectors
Step 4: HYPOTHESIS FORMATION → one testable hypothesis per anomaly
Step 5: EVIDENCE CHAIN VALIDATION → completeness, consistency, reliability

Output: structured reasoning report
```

## Step 1: Clue Extraction

```
From any text, extract:

Regex patterns:
  IP:        \b(?:\d{1,3}\.){3}\d{1,3}\b
  Domain:    \b[a-zA-Z0-9-]+(?:\.[a-zA-Z]{2,})+\b
  CVE:       CVE-\d{4}-\d{4,}
  Port:      port: (\d{2,5})
  Version:   (\d+\.\d+(?:\.\d+)?)
  Email:     \b[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}\b
  URL:       https?://[^\s]+

Semantic patterns:
  Vuln type:   SQLi / XSS / RCE / BoF / LFI / SSRF / CSRF / PrivEsc
  Technique:   ROP / ret2libc / heap spray / format string
  Tool:        nmap / burp / metasploit / sqlmap / gdb

Output format:
  CLUE-{id}: {type}={value} | source={tool/agent} | confidence={0.0-1.0}
```

## Step 2: Cross-Correlation

```
Relation types:

1. Spatial: same host/IP
   CLUE-001(192.168.1.1:80) + CLUE-002(192.168.1.1:443) → same host

2. Causal: A enables B
   Port open → Service running → Version fingerprinted → CVE matched

3. Temporal: same timeframe events

4. Type: same vulnerability class across different instances

Output format:
  REL-{id}: {clue_a} --[{relation}]--> {clue_b}
```

## Step 3: Anomaly Detection (12 detectors)

```
PortAnomaly:         Non-standard ports
ServiceAnomaly:      Outdated/vulnerable service versions
CVEAnomaly:          Version matches known CVE
CredentialAnomaly:   Default/weak credentials
ConfigAnomaly:       Insecure configuration
PatchAnomaly:        Known vulnerability unpatched
ChainAnomaly:        Clues form attack chain (CRITICAL)
TrafficAnomaly:      Unusual network patterns
FileAnomaly:         Unusual filesystem activity
ProcessAnomaly:      Unusual process behavior
LogAnomaly:          Suspicious audit log entries
BehaviorAnomaly:     Deviation from baseline

Output format:
  ANOM-{id}: [{detector}] {description} | SEVERITY: {level}
```

## Step 4: Hypothesis Formation

```
Template:
  H-{id}: {claim}
  Based on: [{clue_ids}]
  Test: {specific test action}
  Expected if true: {what to observe}
  Expected if false: {what to observe}
  Confidence: {0.0-1.0}
```

## Step 5: Evidence Chain Validation

```
For each conclusion:

1. Completeness:
   [ ] Does every Hypothesis have an Observation?
   [ ] Does every Conclusion have a Verification?

2. Consistency:
   [ ] Are there contradictory findings?

3. Reliability:
   [ ] Are sources trusted? (tool output > agent inference > external)

4. Reproducibility:
   [ ] Are verification steps explicit?

Output: PASS or FAIL: {reason} | Confidence: {0.0-1.0}
```

## Reasoning Report Template

```
═══════════════════════════════════
REASONING REPORT
═══════════════════════════════════
Input: {source}
Clues extracted: {n}
  CLUE-001: {type}={value} | {source} | confidence={c}

Correlations: {m}
  REL-001: {a} --[{rel}]--> {b}

Anomalies: {k}
  ANOM-001: [{detector}] {description} | SEVERITY: {level}

Hypotheses: {h}
  H-001: {claim} | Test: {test} | Confidence: {c}

Evidence Chain: {PASS or FAIL} | Overall confidence: {c}
═══════════════════════════════════
```

## Failure Pattern Matching

```
On verification failure:

Match against known patterns:
  "Connection refused" → target unreachable
  "Access denied" → patch applied or wrong credentials
  "Timeout" → firewall or network isolation
  "Unexpected output" → tool version incompatibility

Select recovery:
  Connection refused → check reachability → retry
  Access denied → check patch version → alternative CVE
  Timeout → check firewall → different port/protocol
  Unexpected output → switch tool version → manual verify

Log: [FAIL] {timestamp} | {tool} | {error} | {root_cause} | {recovery} | {success}
```

---
name: secagent-knowledge
description: Local LLM-Wiki knowledge system. Bootstrap a structured wiki, write entries using precise templates, full-text search, cross-agent knowledge sharing, cross-session inheritance.
tools: read_file, write_file, list_dir, grep_files
---

# SecAgent Knowledge — Local LLM-Wiki

Loading this SKILL gives you a complete local knowledge management system. This is not documentation — you actually create and maintain a structured wiki.

## Bootstrap (first load)

```
1. Determine wiki root:
   WIKI_ROOT = {workspace}/wiki

2. Create directory skeleton:
   mkdir -p {WIKI_ROOT}/entities/{ips,domains,cves,tools}
   mkdir -p {WIKI_ROOT}/concepts/{attack-methods,defense-tech,design-patterns}
   mkdir -p {WIKI_ROOT}/discoveries/{scan-results,vuln-findings,pocs}
   mkdir -p {WIKI_ROOT}/synthesis/{reports,analysis,summaries}

3. Initialize INDEX.md at {WIKI_ROOT}/INDEX.md:
   "# Wiki Index\n\nLast updated: {timestamp}\n\n## Entities\n\n## Discoveries\n\n## Synthesis\n"

4. Confirm: "Wiki ready at {WIKI_ROOT}. Starting knowledge production."
```

## The Knowledge Loop: Produce → Structure → Retrieve → Reuse

### Produce (on every discovery)

```
Trigger → Action:
  IP + port + service found           → write entities/ips/{ip}.md
  Product + version identified        → write entities/cves/{cve-id}.md
  CVE matched                         → write entities/cves/{cve-id}.md
  Vulnerability confirmed             → write discoveries/vuln-findings/F{###}.md
  PoC successful                      → write discoveries/pocs/{finding-id}-poc.md
  Sub-agent returns results           → extract findings → write to matching directory
  Attack pattern identified           → write concepts/attack-methods/{technique}.md

File naming: use entity identifiers (IP, CVE ID, Finding ID), not descriptive titles.
After each write: append to the matching section in INDEX.md.
```

### Structure (write with exact templates)

**IP Entry** (`wiki/entities/ips/{ip}.md`):
```markdown
# {ip}
- **First seen**: {timestamp}
- **Last seen**: {timestamp}
- **Hostname**: {hostname or "unknown"}
- **OS**: {os or "unknown"}

## Open Ports
| Port | Service | Version | CVE |
|------|---------|---------|-----|
| {port} | {service} | {version} | {cve or "—"} |

## Findings
- [{finding-id}](../discoveries/vuln-findings/{finding-id}.md): {one-line summary}
```

**CVE Entry** (`wiki/entities/cves/{cve-id}.md`):
```markdown
# {cve-id}
- **Product**: {product} {version}
- **Type**: {sqli/xss/rce/bof/lfi/ssrf/...}
- **CVSS**: {score}
- **CWE**: {cwe-id}
- **Exploit**: {link or "no public exploit"}
- **PoC**: [poc](../discoveries/pocs/{finding-id}-poc.md)
- **Fixed in**: {version or "unknown"}
- **Verified**: {yes/no/untested}
- **Verified against**: {target or "—"}
```

**Finding Entry** (`wiki/discoveries/vuln-findings/F{###}.md`):
```markdown
# F{###}: {title}
- **Target**: [REDACTED]
- **Endpoint**: {url or path}
- **Type**: {vuln type}
- **Severity**: Critical / High / Medium / Low
- **Status**: Confirmed / Pending / False Positive
- **CVSS**: {score or "—"}

## Evidence
{tool output, screenshots, payloads}

## Impact
{concrete impact}

## Reproduction
1. {step 1}
2. {step 2}
3. {expected result}

## Remediation
{specific fix}
```

**Report Template** (`wiki/synthesis/reports/{date}-{engagement}.md`):
```markdown
# {engagement} — Security Assessment Report
**Date**: {date} | **Scope**: {targets} | **Findings**: {count}

## Severity Breakdown
- Critical: {n} | High: {n} | Medium: {n} | Low: {n}

## Findings
{list of F### links with one-line summaries}

## Timeline
{chronological log}

## Recommendations
{prioritized remediation steps}
```

### Retrieve (before starting new work)

```
Search priority — always search BEFORE starting:
  1. grep_files("wiki/", "{keyword}")          → filename search
  2. read_file("wiki/INDEX.md")                → check index
  3. grep_files("wiki/", "{product} {version}")→ precise product search
  4. grep_files("wiki/", "{cve-id}")           → CVE search
  5. read_file("wiki/entities/ips/{target}.md")→ existing target info
```

### Reuse (cross-agent / cross-session)

```
Same session sharing:
  Agent A writes → Agent B reads immediately (same filesystem)
  Protocol: Agent A write → Agent A report path → Agent B read

Cross-session inheritance:
  New session → search wiki/INDEX.md
  → Read entries modified in last 3 days
  → Inject context: "Previous session discovered: {summary}"
  → Continue, don't repeat completed work

Curation (every 10 writes or session end):
  1. Entries not accessed in 90 days → mark [STALE]
  2. Duplicate entries → merge
  3. Update INDEX.md
  4. Report: "Wiki curation: {n} stale, {m} merged, {total} total entries"
```

## Quality Self-Check

After each write:
- [ ] Is the filename searchable? (IP, CVE ID, Finding ID — not descriptive titles)
- [ ] Is the template complete? (all required fields filled?)
- [ ] Are there cross-references? (links to related entries?)
- [ ] Is INDEX.md updated?

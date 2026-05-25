# SecAgent Skills

> Security capability SKILL arsenal — 6 independently-loadable security skill modules for AI agents

Extracted and distilled from [SecAgent-TUI](https://github.com/JesusConwellpy/SecAgent-TUI). Each SKILL is an **executable instruction set** — an AI agent gains the corresponding capability immediately upon loading.

[中文说明](./README.zh-CN.md)

---

## SKILL Index

| SKILL | Capability | Summary |
|-------|-----------|---------|
| [`secagent-identity`](./secagent-identity/SKILL.md) | Identity | Dual-mode persona + reasoning chain + thinking depth |
| [`secagent-methodology`](./secagent-methodology/SKILL.md) | Methodology | 4-phase pentest cycle + CVE matching + defense bypass |
| [`secagent-orchestration`](./secagent-orchestration/SKILL.md) | Orchestration | 9 agent types + fan-out + mailbox communication |
| [`secagent-sandbox`](./secagent-sandbox/SKILL.md) | Sandbox | Approval chain + network ACL + execution audit |
| [`secagent-reasoning`](./secagent-reasoning/SKILL.md) | Reasoning | Clue extraction + causal graph + 12 anomaly detectors |
| [`secagent-knowledge`](./secagent-knowledge/SKILL.md) | Knowledge | Local LLM-Wiki + cross-agent sharing + cross-session memory |

---

## Quick Start

```bash
# Load a single SKILL in Claude Code
/load_skill secagent-identity

# Or read directly
Skill("secagent-methodology")
```

Any AI agent with file-read capability can load a `SKILL.md` directly.

---

## Design Principles

- **Executable, not documentation** — Each SKILL is an instruction set, not an API reference
- **Self-bootstrapping** — Agent initializes its own environment on first load
- **Independent** — Each SKILL works standalone, or combined
- **Universal** — Not tied to any specific language or framework

---

## Source

Extracted from [SecAgent-TUI](https://github.com/JesusConwellpy/SecAgent-TUI) — a deep fork of DeepSeek-TUI built as a multi-agent security research platform.

## License

MIT

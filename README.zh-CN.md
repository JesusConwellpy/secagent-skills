# SecAgent Skills

> 安全攻防能力 SKILL 武器库 — 6 个可被 AI Agent 独立加载的安全技能模块

从 [SecAgent-TUI](https://github.com/JesusConwellpy/SecAgent-TUI) 二开项目中蒸馏提取。每个 SKILL 是**可执行指令集**，AI Agent 加载后即刻获得对应能力。

---

## SKILL 目录

| SKILL | 能力 | 一句话 |
|-------|------|--------|
| [`secagent-identity`](./secagent-identity/SKILL.md) | 安全身份 | 双模人格 + 推理链 + 思维深度控制 |
| [`secagent-methodology`](./secagent-methodology/SKILL.md) | 攻击方法论 | 4 阶段渗透测试 + CVE 匹配 + 防御绕过 |
| [`secagent-orchestration`](./secagent-orchestration/SKILL.md) | 子代理编排 | 9 种 Agent 类型 + Fan-Out + mailbox |
| [`secagent-sandbox`](./secagent-sandbox/SKILL.md) | 安全沙箱 | 审批链 + 网络 ACL + 执行审计 |
| [`secagent-reasoning`](./secagent-reasoning/SKILL.md) | 侦探推理 | 线索提取 + 因果图 + 12 种异常检测 |
| [`secagent-knowledge`](./secagent-knowledge/SKILL.md) | 知识管理 | 本地 LLM-Wiki + 跨 Agent 共享 + 跨会话继承 |

---

## 快速开始

```bash
# Claude Code 中加载单个 SKILL
/load_skill secagent-identity

# 或直接读取
Skill("secagent-methodology")
```

任何支持文件读取的 AI Agent 均可直接加载 `SKILL.md`。

---

## 设计原则

- **可执行，非文档** — 每个 SKILL 是指令集，不是 API 参考
- **自举** — Agent 加载后自行初始化所需环境
- **独立** — 每个 SKILL 可单独加载，也可组合使用
- **通用** — 不绑定特定语言或框架

---

## 来源

提取自 [SecAgent-TUI](https://github.com/JesusConwellpy/SecAgent-TUI) — 基于 DeepSeek-TUI 的深度二开安全研究平台。

## License

MIT

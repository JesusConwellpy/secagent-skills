# SecAgent Skills — 安全攻防能力 SKILL 武器库

从 SecAgent-TUI (JesusConwellpy/SecAgent-TUI) 二开项目中提取的 6 个独立安全能力模块。

每个 SKILL 可被 AI Agent (Claude Code, Codex 等) 通过 `load_skill` 独立加载，获得对应的安全攻防能力。

## SKILL 目录

| SKILL | 能力 | 调用链 |
|-------|------|--------|
| `secagent-identity` | 安全身份与 Prompt 体系 | 模式分类 → 4 层 prompt 组合 → 推理链注入 |
| `secagent-orchestration` | 子代理编排与通信 | 任务分析 → Persona 匹配 → spawn → mailbox → 合成 |
| `secagent-sandbox` | 沙箱执行与安全隔离 | 工具调用 → 审批链 → Landlock/netns → 安全返回 |
| `secagent-methodology` | 攻击方法论与推理链 | 目标 → 4 阶段循环 → CVE 匹配 → 绕过 → 报告 |
| `secagent-knowledge` | 知识管理与 Wiki 系统 | 发现 → Wiki 写入 → FTS5 搜索 → 跨会话复用 |
| `secagent-reasoning` | 侦探推理引擎与因果图 | 线索提取 → 因果图 → 异常检测 → 假设验证 |

## 使用方式

### Claude Code

```bash
# 加载单个 SKILL
/load_skill secagent-identity

# 或直接引用
Skill("secagent-methodology")
```

### 其他 AI Agent

每个 `SKILL.md` 是标准 Markdown 文件，任何支持文件读取的 Agent 均可加载。

## 来源

提取自 [SecAgent-TUI](https://github.com/JesusConwellpy/SecAgent-TUI) — 基于 DeepSeek-TUI 的二开安全研究平台。

## License

MIT

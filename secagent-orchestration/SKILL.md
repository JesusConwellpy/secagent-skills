---
name: secagent-orchestration
description: 安全子代理编排系统 — 9 种子代理类型的自动匹配、spawn、mailbox 通信、结果合成
---

# SecAgent Orchestration — 子代理编排与通信

## 能力概述

管理安全子代理的完整生命周期：任务分析 → Persona 匹配 → 并行 spawn → mailbox 通信 → 结果合成。支持 9 种安全专用子代理类型，自动协调多 Agent 并行工作。

## 调用链

```
安全检查任务
  → selector.rs: 任务分析 → Persona 匹配
    → persona_builder.rs: 定制 Persona 生成
    → agent_spawn(type, prompt):
        → injector.rs: Persona 注入子代理 system prompt
        → 子代理启动 (独立上下文)
    → mailbox.rs: 双向通信
        → agent_progress: 进度更新
        → agent_result: 结构化输出
        → agent_complete: 完成通知 (<secagent:subagent.done>)
    → 协调员: 证据评估 → 结果合成
```

## 9 种子代理类型

| 类型 | Persona 文件 | 能力 |
|------|-------------|------|
| `recon` | personas/recon.md | 端口扫描, 服务发现, OSINT, 子域名枚举 |
| `web-analyst` | personas/web-analyst.md | SQLi/XSS/LFI/SSRF/CSRF 检测与验证 |
| `binary-analyst` | personas/binary-analyst.md | checksec, 反汇编, ROP/shellcode, GDB |
| `exploit-runner` | personas/exploit-runner.md | PoC 执行, 沙箱隔离, 漏洞验证 |
| `intel-gatherer` | personas/intel-gatherer.md | CVE 搜索, exploit-db, GitHub PoC |
| `code-auditor` | personas/code-auditor.md | 源码审计, 漏洞发现, 静态分析 |
| `post-exploit` | personas/post-exploit.md | 权限维持, 横向移动, 信息收集 |
| `coordinator` | personas/coordinator.md | 团队编排, 证据评估, 报告生成 |
| `squad-red/cross-test` | personas/squad-*.md | 红队, 交叉测试 |

## 并行 Fan-Out 规则

```
N 个独立调查 → N 个 agent_spawn 同回合发出
例: 3 targets → 3 recon agents in ONE turn
    + 1 intel-gatherer for CVE lookup
    = 4 agents running in parallel
```

## Mailbox 通信协议

```
agent_spawn    → { id, prompt, type }
agent_progress → { id, status }          // "step 3/30: requesting model response"
agent_complete → { id, result }          // 结构化输出 + summary
```

协调员接收 `<secagent:subagent.done>` sentinel，提取 summary，不做子代理已做的工作。

## Task Spec → Custom Persona Pipeline

```
TaskSpec (来自 agent_spawn prompt)
  → selector::select_persona(task_spec)
    → 匹配最佳基础 Persona
  → persona_builder::build_custom_persona(base_card, task_spec)
    → 生成定制 Persona
  → register_custom_persona(custom)
  → agent_spawn(type=custom)
```

## 关键文件

- `crates/tui/src/persona/selector.rs` — 任务→Persona 匹配器
- `crates/tui/src/persona/persona_builder.rs` — 定制 Persona 生成器
- `crates/tui/src/persona/team_designer.rs` — 团队编排蓝图
- `crates/tui/src/persona/injector.rs` — Persona XML 注入器
- `crates/tui/src/tools/subagent/mod.rs` — SubAgentType, SubAgentResult, SubAgentStatus
- `crates/tui/src/tools/subagent/mailbox.rs` — Agent 通信邮箱
- `crates/tui/assets/personas/` — 9 种 Persona 定义文件

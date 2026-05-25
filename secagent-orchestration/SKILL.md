---
name: secagent-orchestration
description: 安全子代理编排系统 — 9 种 Agent 类型自动匹配、并行 spawn/fan-out、mailbox 双向通信、结构化结果合成
version: "1.0"
source: JesusConwellpy/SecAgent-TUI (crates/tui/src/tools/subagent/, crates/tui/src/persona/)
tools: agent_spawn, agent_wait, agent_result, agent_cancel, agent_list, agent_send_input
---

# SecAgent Orchestration — 子代理编排与通信

## 能力概述

管理安全子代理的完整生命周期。9 种安全专用 Agent 类型按需匹配，Persona Card 自动注入，并行 fan-out 最大化吞吐，mailbox 双向通信标准化，结果结构化合成。

## 调用链 (完整端到端)

```
安全检查任务
  │
  ├─→ Persona 选择 (selector.rs)
  │   select_persona(task_spec) → PersonaCard
  │   │
  │   ├─ 精确匹配: task domain match persona expertise
  │   ├─ 相似匹配: related domain match
  │   └─ 默认分配: General purpose
  │
  ├─→ Persona 定制 (persona_builder.rs)
  │   build_custom_persona(base_card, task_spec) → PersonaCard
  │   register_custom_persona(custom) → registered
  │
  ├─→ Agent Spawn (subagent/mod.rs)
  │   agent_spawn(type, prompt, fork_context)
  │   │
  │   ├─ 构造 SubAgentConfig
  │   ├─ Persona 注入 (injector.rs)
  │   │   → 子代理 system prompt = persona_card.xml + base_prompt
  │   ├─ 子代理进程/线程启动
  │   └─ 返回 agent_id
  │
  ├─→ Mailbox 通信 (mailbox.rs)
  │   │
  │   ├─ MailboxMessage::Progress { agent_id, status }
  │   │   → agent_progress 事件 → AgentState.status 更新
  │   │
  │   ├─ MailboxMessage::Result { agent_id, result }
  │   │   → agent_result 事件 → AgentState.result 更新
  │   │
  │   └─ MailboxMessage::Done { agent_id, summary }
  │       → <secagent:subagent.done> sentinel
  │       → 协调员读取 summary → 合成结论
  │
  ├─→ 并行 Fan-Out
  │   N 个独立任务 → N 个 agent_spawn 同回合发出
  │   例: 3 targets → 3 recon + 1 intel-gatherer = 4 agents parallel
  │
  └─→ 结果合成
      agent_wait(all) → 收集所有结果
      → 交叉验证 (cross-test squad)
      → 合成最终结论
```

## 9 种子代理类型 (完整定义)

```rust
// crates/tui/src/tools/subagent/mod.rs:228
pub enum SubAgentType {
    General,        // 通用 Agent
    Explore,        // 代码库探索 (只读搜索)
    Plan,           // 规划设计
    Review,         // 代码审查
    Implementer,    // 代码实现
    Verifier,       // 验证测试
    ToolAgent,      // 工具调用 Agent
    Custom,         // 自定义角色
}

// crates/tui/src/persona/selector.rs — TeamRole
pub enum TeamRole {
    Recon,           // 侦察: 端口扫描, 服务发现, OSINT
    WebAnalyst,      // Web 分析: SQLi/XSS/LFI/SSRF 检测
    BinaryAnalyst,   // 二进制: checksec, disassemble, ROP/shellcode
    ExploitRunner,   // 漏洞利用: PoC 执行, 沙箱隔离
    IntelGatherer,   // 情报: CVE/exploit-db/GitHub 搜索
    CodeAuditor,     // 代码审计: 源码漏洞发现
    PostExploit,     // 后渗透: 权限维持, 横向移动
    Coordinator,     // 协调: 团队编排, 证据评估
    Custom,          // 自定义
}
```

## 子代理生命周期状态机

```rust
// crates/tui/src/tools/subagent/mod.rs:438
pub enum SubAgentStatus {
    Running,                    // Agent 正在执行
    Completed,                  // 正常完成
    Interrupted(String),        // 被中断 (原因)
    Failed(String),             // 失败 (错误信息)
    Cancelled,                  // 被取消
}

// 状态转换:
//   spawn → Running
//   Running → Completed (正常)
//   Running → Interrupted (外部中断)
//   Running → Failed (内部错误)
//   Running → Cancelled (用户取消)
```

## SubAgentResult (完整结构体)

```rust
// crates/tui/src/tools/subagent/mod.rs:448
pub struct SubAgentResult {
    pub name: String,                       // 可读名称
    pub agent_id: String,                   // UUID
    pub context_mode: String,               // "fork" or "inline"
    pub fork_context: bool,                 // 是否 fork 独立上下文
    pub agent_type: SubAgentType,           // 类型枚举
    pub assignment: SubAgentAssignment,     // { objective, role }
    pub model: String,                      // 使用的模型
    pub nickname: Option<String>,           // 昵称
    pub status: SubAgentStatus,             // 当前状态
    pub result: Option<String>,             // 结果输出
    pub steps_taken: u32,                   // 执行步数
    pub duration_ms: u64,                   // 耗时 (毫秒)
    pub from_prior_session: bool,           // 是否跨会话
}
```

## SubAgentAssignment

```rust
// crates/tui/src/tools/subagent/mod.rs
pub struct SubAgentAssignment {
    pub objective: String,    // 任务目标 (一句话)
    pub role: String,         // 角色描述
}
```

## Fan-Out 规则 (并行编排)

```
铁律: N 个独立调查 → N 个 agent_spawn 同回合发出

正确:
  Turn 1: spawn recon(A) + spawn recon(B) + spawn intel-gatherer(apache CVEs)
  → All 3 run in parallel. Results come back as subagent.done events.

错误:
  Turn 1: spawn recon(A)
  Turn 2: wait → spawn recon(B)
  → Serial execution wastes time. Parallel is always faster.

Fan-Out 触发条件:
  - >1 target in scope → spawn recon per target IN PARALLEL
  - Product+version identified, no CVE search done → spawn intel-gatherer NOW
  - Hypothesis score >= 7 → spawn specialist for verification
  - Exploitation path clear → spawn exploit-runner
  - Any binary file provided → spawn binary-analyst
```

## Mailbox 通信协议

```rust
// crates/tui/src/tools/subagent/mailbox.rs
pub enum MailboxMessage {
    Progress {
        agent_id: String,
        status: String,     // "step 3/30: requesting model response"
    },
    Result {
        agent_id: String,
        result: String,     // 结构化子代理输出
    },
    Done {
        agent_id: String,
        summary: String,    // 1-3 句摘要
    },
    Input {
        agent_id: String,
        prompt: String,     // 主 Agent 向子代理发送输入
    }
}
```

### 协调员消费模式

```
1. agent_spawn(type="recon", prompt="scan target A")
2. 等待 <secagent:subagent.done> sentinel
3. 读取 summary 字段 (不要重新做子代理已做的工作)
4. 调用 agent_eval(agent_id) 拉取完整结构化输出
5. 集成到主工作流
```

## Team Designer (团队蓝图)

```rust
// crates/tui/src/persona/team_designer.rs
pub struct TeamBlueprint {
    pub name: String,                  // "Squad-Red" for a pentest engagement
    pub objective: String,             // "Extract admin credentials from target"
    pub members: Vec<TeamMember>,
}

pub struct TeamMember {
    pub persona: String,               // "recon", "web-analyst", etc.
    pub count: usize,                  // 启动几个实例
    pub assignment: String,            // 该成员的具体任务
    pub depends_on: Vec<String>,       // 必须等待哪些成员先完成
}

pub fn design_team(objective: &str, personas: &[PersonaCard]) -> TeamBlueprint
pub fn render_blueprint(blueprint: &TeamBlueprint) -> String
```

## Agent 工具集

```
agent_spawn(type, prompt, fork_context?) → agent_id
  Spawn a new sub-agent with optional context forking.

agent_wait(agent_ids[]) → Vec<SubAgentResult>
  Block until specified agents complete.

agent_result(agent_id) → SubAgentResult
  Pull structured result for a specific agent.

agent_cancel(agent_id)
  Cancel a running agent.

agent_list() → Vec<AgentState>
  List all spawned agents with status.

agent_send_input(agent_id, input)
  Send stdin input to a running agent.

agent_assign(agent_id, assignment)
  Reassign an agent's task mid-execution.

resume_agent(agent_id)
  Resume a previously paused/interrupted agent.
```

## 集成示例

### 简单侦察

```
User: "scan example.com for web vulnerabilities"

Agent:
  1. spawn agent_spawn type="recon" prompt="Enumerate example.com: ports, subdomains, services"
  2. spawn agent_spawn type="web-analyst" prompt="Map web surface on example.com"
  3. agent_wait(all) — both run in parallel
  4. 评估结果 → 如果 SQL error found → spawn web-analyst for SQLi verification
```

### CTF 解题

```
User: "solve this pwn challenge binary"

Agent:
  1. spawn agent_spawn type="binary-analyst" prompt="Analyze binary: checksec, strings, disassemble main()"
  2. agent_wait(agent_id)
  3. 结果: "NX enabled, no PIE, no canary, gets() at 0x401234"
  4. OBSERVE → HYPOTHESIZE: "classic BOF, EIP control likely at offset"
  5. spawn agent_spawn type="binary-analyst" prompt="Find buffer offset to EIP. Build ROP chain."
  6. agent_wait → flag captured
```

## 关键文件 (源仓库)

| 文件 | 行数 | 用途 |
|------|------|------|
| `crates/tui/src/tools/subagent/mod.rs` | 600+ | 子代理核心 (类型, 状态, 结果) |
| `crates/tui/src/tools/subagent/mailbox.rs` | 200+ | Mailbox 通信协议 |
| `crates/tui/src/persona/selector.rs` | 80+ | 任务→Persona 匹配 |
| `crates/tui/src/persona/persona_builder.rs` | 60+ | 定制 Persona 生成 |
| `crates/tui/src/persona/team_designer.rs` | 80+ | 团队蓝图设计 |
| `crates/tui/src/persona/injector.rs` | 75 | Persona 注入器 |
| `crates/tui/assets/personas/` | 10 文件 | Persona 定义 (Markdown) |

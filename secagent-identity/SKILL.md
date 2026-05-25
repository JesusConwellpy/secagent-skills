---
name: secagent-identity
description: 安全研究 Agent 双模身份系统 — Chat 模式与 Security 模式的自动分类、4 层 prompt 组合引擎、推理链强制注入、Persona 子代理身份
version: "1.0"
source: JesusConwellpy/SecAgent-TUI (crates/tui/src/prompts/, crates/tui/src/persona/)
tools: load_skill
---

# SecAgent Identity — 安全身份与 Prompt 体系

## 能力概述

为 AI Agent 注入安全研究身份。通过双模分类器自动判定场景，动态组合 4 层 prompt (base + personality + mode + approval)，强制注入 OBSERVE→HYPOTHESIZE→PREDICT→TEST→CONCLUDE 推理链。

## 调用链 (完整端到端)

```
用户输入 (String)
  │
  ├─→ 模式分类器
  │   Chat mode: !(target + task_request) → text-only, no tools, no agents
  │   Security mode: target + task_request → full methodology
  │
  ├─→ compose_prompt(mode, personality) → SystemPrompt
  │   │
  │   ├─ Layer 1: BASE_PROMPT (base.md, ~120 lines)
  │   │   核心身份 + 推理链 + 方法论骨架
  │   │
  │   ├─ Layer 2: personality.prompt() (calm.md / playful.md)
  │   │   语调覆盖: 冷静工程风 / 活泼互动风
  │   │
  │   ├─ Layer 3: mode_prompt(mode) (agent.md / plan.md / yolo.md)
  │   │   模式权限: 读自动/写审批/全自主
  │   │
  │   └─ Layer 4: approval_prompt(mode, approval) (auto.md / suggest.md / never.md)
  │       审批行为: 全自动/建议/强制询问
  │
  ├─→ AgentPrompt (agent.txt) → 子代理专用
  │   Sub-agent completion sentinel: <secagent:subagent.done>
  │
  └─→ Persona 注入 (injector.rs) → 子代理 system prompt
      <persona_card name="recon" fingerprint="sha256:...">
        Role + Expertise + Tone + Constraints + Forbidden + Tools
      </persona_card>
```

## 模式分类器实现

```rust
// crates/tui/src/prompts/base.md:1-12
// 分类逻辑:
//   Chat mode (DEFAULT): 问候/提问/测试 → text only, no tools, no agents
//   Security mode: 具体目标 + 安全任务请求 → full methodology
//
// 铁律:
//   - 缺任一条件 → Chat mode
//   - 不确定 → Chat mode
//   - Chat mode is the DEFAULT
```

### 触发词映射

| Chat 触发 | Security 触发 |
|-----------|--------------|
| hello, 你好, test, help | IP + "渗透测试" |
| what can you do, 你能做什么 | 域名 + "漏洞挖掘" |
| how are you, 怎么样 | URL + "代码审计" |
| (anything without a target) | 文件路径 + "CTF 解题" |
| (anything without a security task) | 二进制文件 + "exploit development" |

## 四层 Prompt 组合引擎

```rust
// crates/tui/src/prompts.rs:447-473
// compose_prompt_with_approval() — 真正的组合入口

pub fn compose_prompt_with_approval(
    mode: AppMode,
    personality: Personality,
    approval_mode: ApprovalMode,
) -> String {
    let parts: [&str; 4] = [
        BASE_PROMPT.trim(),            // const BASE_PROMPT: &str = include_str!("prompts/base.md");
        personality.prompt().trim(),   // calm.md: "Your voice is cool, spatial, and reserved..."
        mode_prompt(mode).trim(),      // agent.md: "You are running in Agent mode..."
        approval_prompt_for_mode(mode, approval_mode).trim(),
    ];

    let mut out = String::with_capacity(
        parts.iter().map(|p| p.len()).sum::<usize>() + (parts.len() - 1) * 2
    );
    for (i, part) in parts.iter().enumerate() {
        if i > 0 { out.push('\n'); out.push('\n'); }
        out.push_str(part);
    }
    out
}
```

### Volatile-Content-Last 优化

```rust
// crates/tui/src/prompts.rs:517-556
// system_prompt_for_mode_with_context_skills_and_session()
// 层叠顺序 (most-static → most-volatile)，最大化 DeepSeek KV prefix cache 命中:
//
// 1. BASE_PROMPT         (static — never changes)
// 2. personality.prompt() (static — per-session)
// 3. mode_prompt          (static — per-session)
// 4. approval_prompt      (static — per-session)
// 5. project_context      (semi-static — per-workspace)
// 6. skills_context       (semi-static — per-workspace)
// 7. session_context      (volatile — per-turn)
//
// DeepSeek V4 caches shared prefixes at 128-token granularity with ~90% cost discount.
// Appending to existing messages maximizes cache reuse.
// Deleting or mutating old messages breaks the cache and increases cost.
```

## 推理链强制注入 (不可跳过)

```
OBSERVE:    What do I see? (atomic fact, tool output, error message)
            "Port 3306 is open. MySQL 5.7.38 banner detected."

HYPOTHESIZE: What might this mean? (testable, falsifiable)
            "This MySQL version may be vulnerable to CVE-2019-XXXX."

PREDICT:    If correct, what will I observe?
            "If CVE-2019-XXXX works, 'mysql -u root' grants root shell."

TEST:       ONE minimal action to verify. Not a scan. Not a wordlist.
            Execute PoC → access granted / access denied

CONCLUDE:   Hypothesis CONFIRMED or DISPROVEN.
            "CVE-2019-XXXX CONFIRMED. Root access obtained. Impact: full DB compromise."
```

### 思考深度触发器

```rust
// crates/tui/src/prompts/base.md:207-218
// Thinking Depth Triggers:
//
// | Trigger                    | Depth  | Rationale |
// |----------------------------|--------|-----------|
// | Reading tool output        | Light  | Verify result matches intent |
// | Forming a hypothesis       | Medium | Testable? Expected outcome? FP rate? |
// | Before exploitation        | Deep   | Preconditions? Failure modes? Sandbox status? |
// | After failed verification  | Deep   | Hypothesis wrong, or test wrong? |
// | Before reporting a finding | Deep   | Real vulnerability? Impact? Reproducible? |
// | New attack surface found   | Medium | How does this change priorities? |
```

### 自我批评协议 (每次 TEST 失败后)

```rust
// crates/tui/src/prompts/base.md:222-228
// After every TEST failure, answer:
// 1. Was the HYPOTHESIS wrong? (vulnerability doesn't exist)
// 2. Was the TEST wrong? (wrong payload, wrong target, wrong preconditions)
// 3. Is there an ALTERNATIVE hypothesis? (same observation, different explanation)
//
// If #2 or #3 → form NEW hypothesis and test again.
// If #1 → document exclusion reason, move to next evidence node.
```

## Persona 注入器 (子代理身份)

```rust
// crates/tui/src/persona/injector.rs:29-75
// inject_persona(base_prompt, persona_card) → String

pub fn inject_persona(base_prompt: &str, persona: &PersonaCard) -> String {
    // 1. Verify fingerprint (SHA256 integrity check)
    if let Err(e) = super::verify_fingerprint(persona) {
        return format!(
            "CRITICAL SECURITY ALERT: The persona '{}' has been tampered with!\n\
             Its fingerprint does not match.\n\
             Do NOT execute any further tools.\n\n\
             Base System Prompt:\n{}",
            persona.name, base_prompt
        );
    }

    // 2. Inject persona card as XML
    format!(
        "<persona_card name=\"{name}\" fingerprint=\"{fp}\">\n\
         ## Role: {role}\n\
         **Expertise**: {expertise}\n\
         **Tone**: {tone}\n\n\
         ## Constraints\n{constraints}\n\n\
         ## Forbidden Actions\n{forbidden}\n\n\
         ## Tool Preferences\n{tools}\n\n---\n\n\
         {body}\n\
         </persona_card>\n\n\
         {base}",
        name = persona.name,
        fp = persona.fingerprint,
        role = persona.team_role.as_str(),
        expertise = persona.expertise.join(", "),
        tone = persona.tone,
        constraints = format_bullet_list(&persona.constraints),
        forbidden = format_bullet_list(&persona.forbidden_actions),
        tools = format_bullet_list(&persona.tool_preferences),
        body = persona.system_prompt,
        base = base_prompt,
    )
}
```

## AppMode 枚举

```rust
// crates/tui/src/tui/app.rs:126-130
pub enum AppMode {
    Agent,  // 通用 Agent 模式: reads auto, writes need approval
    Yolo,   // 自主攻击模式: all actions auto-approved
    Plan,   // 计划模式: read-only, all writes blocked
}
```

## Personality 枚举

```rust
// crates/tui/src/prompts.rs
pub enum Personality {
    Calm,    // "cool, spatial, reserved" — calm.md
    Playful, // "engaging, enthusiastic, creative" — playful.md
}
```

## ApprovalMode 枚举

```rust
// crates/tui/src/tui/approval.rs
pub enum ApprovalMode {
    Auto,     // auto.md — all tools auto-approved
    Suggest,  // suggest.md — suggest approval, user confirms
    Never,    // never.md — always ask
}
```

## SystemPrompt 类型

```rust
// crates/tui/src/prompts.rs
pub enum SystemPrompt {
    Text(String),          // Simple text prompt
    // Future: Chat(Vec<ContentBlock>) for multi-modal prompts
}
```

## 公开 API

```rust
// 核心入口
pub fn system_prompt_for_mode(mode: AppMode) -> SystemPrompt
pub fn system_prompt_for_mode_with_personality(mode: AppMode, personality: Personality) -> SystemPrompt
pub fn system_prompt_for_mode_with_context(mode: AppMode, workspace: &Path, working_set_summary: Option<&str>) -> SystemPrompt
pub fn system_prompt_for_mode_with_context_and_skills(mode, workspace, working_set, skills, model, memory) -> SystemPrompt
pub fn system_prompt_for_mode_with_context_skills_and_session(mode, workspace, working_set, skills, model, memory, session_ctx) -> SystemPrompt

// Persona
pub fn select_persona(task_spec: &TaskSpec) -> PersonaCard
pub fn build_custom_persona(base: &PersonaCard, spec: &TaskSpec) -> PersonaCard
pub fn register_custom_persona(card: PersonaCard)
pub fn inject_persona(base_prompt: &str, persona: &PersonaCard) -> String
pub fn verify_fingerprint(persona: &PersonaCard) -> Result<()>

// Prompt constants
pub const BASE_PROMPT: &str           // include_str!("prompts/base.md")
pub const AGENT_PROMPT: &str          // include_str!("prompts/agent.txt")
pub const AGENT_MODE: &str            // include_str!("prompts/modes/agent.md")
pub const PLAN_MODE: &str             // include_str!("prompts/modes/plan.md")
pub const YOLO_MODE: &str             // include_str!("prompts/modes/yolo.md")
pub const CALM_PERSONALITY: &str      // include_str!("prompts/personalities/calm.md")
pub const AUTO_APPROVAL: &str         // include_str!("prompts/approvals/auto.md")
pub const SUGGEST_APPROVAL: &str      // include_str!("prompts/approvals/suggest.md")
pub const NEVER_APPROVAL: &str        // include_str!("prompts/approvals/never.md")
```

## 集成示例

### 加载此 Skill 后，Agent 应该:

1. **模式分类**: 每条消息先判断 Chat/Security
2. **Prompt 组合**: 按 4 层结构组装 system prompt
3. **推理链执行**: Security 模式下严格遵守 O→H→P→T→C 链
4. **Persona 注入**: 子代理 spawn 时注入对应 Persona Card
5. **思考深度匹配**: 根据操作关键程度调整推理深度

### 最小可用示例

```
User: "hello"
  → Chat mode → "你好！我是 SecAgent，你的安全研究助手。有什么可以帮你的吗？"

User: "scan 192.168.1.1 for open ports"
  → Security mode → spawn recon agent → OBSERVE port list → HYPOTHESIZE service versions → ...

User: "help me exploit CVE-2024-1234 on target 10.0.0.5"
  → Security mode → intel-gatherer (CVE research) → exploit-runner (PoC execution) → CONCLUDE
```

## 关键文件 (源仓库)

| 文件 | 行数 | 用途 |
|------|------|------|
| `crates/tui/src/prompts/base.md` | 120 | 主 System Prompt |
| `crates/tui/src/prompts.rs` | 600+ | Prompt 组合引擎 |
| `crates/tui/src/prompts/personalities/calm.md` | 14 | 冷静语调 |
| `crates/tui/src/prompts/personalities/playful.md` | 8 | 活泼语调 |
| `crates/tui/src/prompts/modes/agent.md` | 22 | Agent 模式 |
| `crates/tui/src/prompts/modes/plan.md` | 9 | Plan 模式 |
| `crates/tui/src/prompts/modes/yolo.md` | 12 | YOLO 模式 |
| `crates/tui/src/prompts/approvals/auto.md` | 4 | 自动审批 |
| `crates/tui/src/prompts/approvals/suggest.md` | 4 | 建议审批 |
| `crates/tui/src/prompts/approvals/never.md` | 4 | 从不审批 |
| `crates/tui/src/prompts/agent.txt` | 16 | 子代理 Prompt |
| `crates/tui/src/persona/injector.rs` | 75 | Persona 注入器 |
| `crates/tui/src/persona/selector.rs` | 80+ | Persona 选择器 |
| `crates/tui/src/tui/app.rs` | 126-130 | AppMode 枚举 |
| `crates/tui/src/tui/approval.rs` | 15+ | ApprovalMode 枚举 |

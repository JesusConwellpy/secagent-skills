---
name: secagent-identity
description: 安全研究 Agent 双模身份系统 — Chat 模式 (纯文本助手) 与 Security 模式 (攻防协调员) 的自动分类、prompt 组合引擎、推理链注入
---

# SecAgent Identity — 安全身份与 Prompt 体系

## 能力概述

为 AI Agent 注入安全研究身份。通过双模分类器自动判定当前场景，动态组合 4 层 prompt (base + personality + mode + approval)，注入 OBSERVE→HYPOTHESIZE→PREDICT→TEST→CONCLUDE 推理链。

## 调用链

```
用户输入
  → 模式分类器 (Chat / Security)
    → Chat: 纯文本助手身份, 无工具无 Agent
    → Security: 安全协调员身份, 4 层 prompt 组合
      → Persona 选择 (Calm/Playful)
      → Mode 注入 (Agent/Plan/Yolo)
      → Approval 策略 (Auto/Suggest/Never)
      → 推理链强制注入 (OBSERVE → HYPOTHESIZE → PREDICT → TEST → CONCLUDE)
  → 最终 system prompt 输出
```

## 核心组件

### 1. 模式分类器

```
Chat 模式 (DEFAULT):
  触发: 问候、提问、测试、闲聊
  行为: 纯文本响应。无工具。无 Agent。无安全方法论。
  关键词: "hello", "你好", "test", "help", "what can you do"

Security 模式:
  触发: 具体目标 (IP/域名/URL/文件路径) + 安全任务请求 (渗透测试/漏洞挖掘/CTF/代码审计)
  行为: 全工具可用。Agent 编排。4 阶段方法论。
```

分类规则: 缺任一条件 → Chat 模式。不确定 → Chat 模式。

### 2. 四层 Prompt 组合

```rust
// compose_prompt() — crates/tui/src/prompts.rs:447
fn compose_prompt(mode: AppMode, personality: Personality) -> String {
    [
        BASE_PROMPT,          // 核心身份 + 方法论
        personality.prompt(), // 语调覆盖
        mode_prompt(mode),    // 模式特定权限
        approval_prompt(),    // 审批策略
    ].join("\n\n")
}
```

### 3. 推理链 (OBSERVE → HYPOTHESIZE → PREDICT → TEST → CONCLUDE)

每个安全操作强制走 5 步推理链。跳过任一步 = 猜测。猜测不是渗透测试。

### 4. 思考深度触发器

| 触发器 | 深度 | 理由 |
|--------|------|------|
| 读取工具输出 | Light | 验证结果匹配意图 |
| 形成假设 | Medium | 可测试性 + 误报率评估 |
| 漏洞利用前 | Deep | 前置条件 + 失败模式 + 沙箱状态 |
| 验证失败后 | Deep | 假设错了还是测试错了？ |
| 报告发现前 | Deep | 真实漏洞？影响范围？可复现？ |

## 子代理 Prompt 注入

```
// inject_persona() — crates/tui/src/persona/injector.rs:29
<persona_card name="recon" fingerprint="sha256:...">
  ## Role: Reconnaissance Specialist
  **Expertise**: 端口扫描, 服务发现, OSINT
  **Tone**: 冷静, 精确
  ## Constraints: 只做侦察, 不执行利用
  ## Forbidden Actions: 禁止利用, 禁止横向移动
  ## Tool Preferences: nmap, curl, whois
</persona_card>
```

## 关键文件

- `crates/tui/src/prompts/base.md` — 主 System Prompt (120 行)
- `crates/tui/src/prompts/personalities/` — 语调定义 (calm.md, playful.md)
- `crates/tui/src/prompts/modes/` — 模式权限 (agent.md, plan.md, yolo.md)
- `crates/tui/src/prompts/approvals/` — 审批策略 (auto.md, suggest.md, never.md)
- `crates/tui/src/prompts/agent.txt` — 子代理 Prompt 模板
- `crates/tui/src/persona/injector.rs` — Persona XML 注入器
- `crates/tui/src/persona/selector.rs` — 任务 → Persona 匹配器
- `crates/tui/src/prompts.rs:447` — compose_prompt() 入口

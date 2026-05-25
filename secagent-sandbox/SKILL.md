---
name: secagent-sandbox
description: 安全沙箱执行系统 — Landlock 文件系统 ACL、netns 网络隔离、rlimits 资源限制、exec policy 审批链、Seatbelt/OpenSandbox 跨平台支持
version: "1.0"
source: JesusConwellpy/SecAgent-TUI (crates/tui/src/sandbox/, crates/execpolicy/, crates/tui/src/network_policy.rs)
tools: exec_shell, task_shell_start, task_shell_wait, sandboxed_exec, validate_data
---

# SecAgent Sandbox — 沙箱执行与安全隔离

## 能力概述

为 AI Agent 的工具调用提供多层安全隔离。exec policy 审批链拦截高风险操作，Landlock + netns + rlimits 形成三道运行时防线。支持 Linux (Landlock/netns)、macOS (Seatbelt)、Windows。

## 调用链 (完整端到端)

```
工具调用请求
  │
  ├─→ Exec Policy Engine (crates/execpolicy/)
  │   │
  │   ├─ ExecPolicy::check(tool_name, params) → AskForApproval
  │   │
  │   ├─ Bash 参数分析 (bash_arity.rs)
  │   │   BashArityDict: 预定义危险 flag 字典
  │   │   检测: --eval, -c, $(...), backticks, IFS manipulation
  │   │
  │   ├─ 规则匹配 (matcher.rs + rules.rs)
  │   │   匹配策略规则: tool_name + param_pattern + risk_level
  │   │
  │   └─ 决策输出 (decision.rs)
  │       → Approve: 放行
  │       → Deny: 拒绝
  │       → Ask: 请求用户审批
  │
  ├─→ 审批框架 (approval.rs)
  │   │
  │   ├─ Auto:   所有工具自动批准
  │   ├─ Suggest: 建议审批 (高风险操作暂停)
  │   └─ Never:  从不自动批准 (强制人工确认)
  │
  ↓ 通过审批
  │
  ├─→ 沙箱包装
  │   │
  │   ├─ Linux:
  │   │   ├─ Landlock (landlock.rs)
  │   │   │   LandlockRuleset:
  │   │   │     .allow_read("/usr")
  │   │   │     .allow_read_write("/tmp")
  │   │   │     .allow_read_write(workspace)
  │   │   │     .deny_all_else()
  │   │   │
  │   │   ├─ netns (backend.rs)
  │   │   │   创建独立网络命名空间
  │   │   │   仅允许 loopback (127.0.0.1)
  │   │   │   可选: 白名单外部 IP (授权目标)
  │   │   │
  │   │   └─ rlimits (backend.rs)
  │   │       RLIMIT_CPU:    CPU 时间限制
  │   │       RLIMIT_AS:     内存地址空间
  │   │       RLIMIT_FSIZE:  文件大小
  │   │       RLIMIT_NPROC:  进程数
  │   │       RLIMIT_NOFILE: 文件描述符
  │   │
  │   ├─ macOS:
  │   │   └─ Seatbelt (seatbelt.rs)
  │   │       sandbox-exec profile:
  │   │         (allow default)
  │   │         (deny file-write* (subpath "/System"))
  │   │         (allow file-write* (subpath workspace) (subpath "/tmp"))
  │   │         (deny network*)
  │   │
  │   └─ 跨平台:
  │       └─ OpenSandbox (opensandbox.rs)
  │           统一抽象层: SandboxBackend trait
  │
  ├─→ 工具执行 (沙箱内)
  │   输出 stdout/stderr → 捕获
  │
  └─→ 结果返回 (沙箱内)
      退出码 + stdout + stderr + 执行时间
```

## Exec Policy Engine

### 策略规则定义

```rust
// crates/execpolicy/src/rule.rs
pub struct ExecPolicyRule {
    pub tool_name: String,          // "exec_shell", "write_file", etc.
    pub param_pattern: Regex,       // 危险参数模式
    pub risk_level: RiskLevel,      // Low / Medium / High / Critical
    pub action: PolicyAction,       // Allow / Deny / Ask
}

// crates/execpolicy/src/rules.rs
pub struct RuleSet {
    pub rules: Vec<ExecPolicyRule>,
    pub default_action: PolicyAction,  // 未匹配规则的默认行为
}
```

### Bash 参数分析

```rust
// crates/execpolicy/src/bash_arity.rs
pub struct BashArityDict {
    // 预定义的危险模式:
    //   --eval, -e — 代码求值
    //   -c — 内联命令
    //   $(...) / backticks — 命令替换
    //   IFS manipulation — 环境变量攻击
    //   /dev/tcp — 反向 shell
    //   nc -e, ncat -e — Netcat 反向 shell
    //   >/dev/null 2>&1 — 输出抑制 (可疑)
}

impl BashArityDict {
    pub fn check(command: &str) -> Vec<DangerFlag>
    pub fn has_danger_flag(command: &str, flag: &str) -> bool
}
```

### 审批决策流程

```rust
// crates/execpolicy/src/decision.rs
pub enum AskForApproval {
    OnRequest,    // 首次请求时询问
    OnModify,     // 修改时询问
    OnCreate,     // 创建时询问
    OnDelete,     // 删除时询问
    Always,       // 始终询问
}

// crates/execpolicy/src/execpolicycheck.rs
pub fn check_exec_policy(
    tool_name: &str,
    params: &HashMap<String, String>,
    rules: &RuleSet,
) -> PolicyResult {
    // 1. 匹配规则
    // 2. 评估风险等级
    // 3. 返回决策
}
```

## Landlock 沙箱 (Linux 5.13+)

```rust
// crates/tui/src/sandbox/landlock.rs
pub struct LandlockRuleset {
    allowed_reads: Vec<PathBuf>,
    allowed_writes: Vec<PathBuf>,
    allowed_execs: Vec<PathBuf>,
}

impl LandlockRuleset {
    pub fn new() -> Self
    pub fn allow_read(mut self, path: &str) -> Self
    pub fn allow_read_write(mut self, path: &str) -> Self
    pub fn allow_exec(mut self, path: &str) -> Self
    pub fn deny_all_else(self) -> Self

    pub fn enforce(self) -> Result<()> {
        // 调用 Linux Landlock ABI:
        //   1. landlock_create_ruleset()
        //   2. landlock_add_rule() × N
        //   3. landlock_restrict_self()
        //
        // 限制规则:
        //   读: 仅白名单路径
        //   写: 仅 workspace + /tmp
        //   执行: 仅白名单二进制
        //   网络: 由 netns 层处理
    }
}
```

### 默认 Landlock 策略

```
Read access:  /usr/ (system libs), /lib/ (system libs), workspace/
Write access: workspace/, /tmp/
Exec access:  (none — no binary execution unless explicitly allowed)
Network:      (none — handled by netns layer)
```

## SandboxBackend Trait (跨平台抽象)

```rust
// crates/tui/src/sandbox/backend.rs
pub trait SandboxBackend: Send + Sync {
    fn name(&self) -> &str;                    // "landlock", "seatbelt", "opensandbox"
    fn setup(&mut self) -> Result<()>;         // 初始化沙箱
    fn restrict_filesystem(&mut self) -> Result<()>;  // 限制文件系统
    fn restrict_network(&mut self) -> Result<()>;     // 限制网络
    fn set_rlimits(&mut self) -> Result<()>;          // 设置资源限制
    fn teardown(&mut self) -> Result<()>;             // 清理
}
```

## 网络访问控制

```rust
// crates/tui/src/network_policy.rs
pub struct NetworkPolicyDecider {
    pub allowed_cidrs: Vec<IpNet>,           // 白名单 CIDR
    pub allowed_domains: Vec<String>,        // 白名单域名 (支持通配符)
    pub session_approvals: Vec<String>,      // 会话级审批 (运行时添加)
    pub audit_log: Vec<NetworkAccessLog>,    // 审计日志
    pub default_deny: bool,                  // 默认拒绝
}

impl NetworkPolicyDecider {
    pub fn with_default_audit(config: NetworkPolicyConfig) -> Self
    pub fn check_access(&self, target: &str) -> PolicyResult
    pub fn approve_host(&mut self, host: &str)  // /network allow <host>
    pub fn audit(&self) -> &[NetworkAccessLog]
}
```

### 授权白名单配置

```toml
# ~/.secagent/authorization.toml
[[authorized]]
cidr = "192.168.0.0/16"
description = "Internal lab network"

[[authorized]]
domain = "*.ctfhub.com"
description = "CTFHub competition platform"

[[authorized]]
domain = "*.hackthebox.com"
description = "HackTheBox"

[[authorized]]
domain = "*.tryhackme.com"
description = "TryHackMe"
```

## 执行策略引擎集成

```rust
// crates/execpolicy/src/policy.rs
pub struct ExecPolicyEngine {
    pub rules: RuleSet,
    pub approval_mode: ApprovalMode,
    pub sandbox_enabled: bool,
}

impl ExecPolicyEngine {
    pub fn check(&self, tool: &str, params: &[String]) -> PolicyResult {
        // 1. Bash 参数扫描 → 危险 flag 列表
        // 2. 规则匹配 → 风险等级
        // 3. 审批模式覆盖 → Auto/Suggest/Never
        // 4. 沙箱状态检查 → 是否已隔离
        // 5. 返回决策
    }
}
```

## 集成示例

### 安全 Shell 执行

```
Agent: exec_shell("nmap -sV 192.168.1.1")

1. ExecPolicyEngine::check("exec_shell", ["nmap", "-sV", "192.168.1.1"])
   → BashArityDict: 无危险 flag → RiskLevel::Medium
   → 规则: exec_shell + 网络工具 → Ask
   → 审批: Suggest → user confirms

2. SandboxBackend::setup()
   → Landlock: RW /tmp + workspace, deny else
   → netns: loopback only
   → rlimits: CPU 30s, AS 256MB, NPROC 10

3. 执行: nmap -sV 192.168.1.1
   → stdout: PORT  STATE SERVICE VERSION
             80/tcp open  http    Apache 2.4.51

4. SandboxBackend::teardown()
   → 释放 Landlock ruleset, 销毁 netns, 恢复 rlimits
```

## 关键文件 (源仓库)

| 文件 | 行数 | 用途 |
|------|------|------|
| `crates/tui/src/sandbox/landlock.rs` | 100+ | Landlock 规则引擎 |
| `crates/tui/src/sandbox/backend.rs` | 150+ | 跨平台沙箱后端 |
| `crates/tui/src/sandbox/seatbelt.rs` | 80+ | macOS Seatbelt |
| `crates/tui/src/sandbox/opensandbox.rs` | 60+ | OpenSandbox 抽象 |
| `crates/tui/src/sandbox/mod.rs` | 50+ | 沙箱模块入口 |
| `crates/tui/src/sandbox/policy.rs` | 40+ | 沙箱策略 |
| `crates/execpolicy/src/policy.rs` | 50+ | ExecPolicyEngine |
| `crates/execpolicy/src/bash_arity.rs` | 60+ | Bash 参数分析 |
| `crates/execpolicy/src/matcher.rs` | 40+ | 策略匹配器 |
| `crates/execpolicy/src/rule.rs` | 30+ | 规则定义 |
| `crates/execpolicy/src/rules.rs` | 30+ | 规则集合 |
| `crates/execpolicy/src/decision.rs` | 20+ | 审批决策 |
| `crates/tui/src/network_policy.rs` | 200+ | 网络访问控制 |
| `~/.secagent/authorization.toml` | — | 授权白名单 |

---
name: secagent-sandbox
description: 安全沙箱执行系统 — 工具调用审批链、Landlock 文件系统隔离、netns 网络隔离、rlimits 资源限制、Seatbelt/OpenSandbox 跨平台支持
---

# SecAgent Sandbox — 沙箱执行与安全隔离

## 能力概述

为 AI Agent 的工具调用提供多层安全隔离。审批框架拦截高风险操作，Landlock + netns + rlimits 形成三层运行时沙箱。支持 Linux (Landlock/netns)、macOS (Seatbelt)、Windows (Windows Sandbox)。

## 调用链

```
工具调用请求
  → exec policy engine (AskForApproval)
    → Auto: 直接放行
    → Suggest: 建议审批
    → Never: 强制拒绝
  ↓ 通过审批
  → Bash 参数分析 (BashArityDict)
    → 危险参数检测
  ↓ 参数安全
  → 沙箱包装
    → Linux: Landlock (文件系统 ACL) + netns (网络隔离) + rlimits (资源限制)
    → macOS: Seatbelt (沙箱 profile)
    → Windows: Windows Sandbox
    → 跨平台: OpenSandbox 抽象层
  → 工具执行
  → 结果返回 (沙箱内)
```

## 三层安全隔离

### 第一层: 审批策略引擎

```
crates/execpolicy/
├── policy.rs      — ExecPolicy, AskForApproval 决策
├── matcher.rs     — 策略匹配器
├── parser.rs      — 策略规则解析
├── rule.rs        — 单条规则定义
├── rules.rs       — 规则集合
├── decision.rs    — 审批决策
├── bash_arity.rs  — Bash 参数分析 (检测危险 flag)
└── execpolicycheck.rs — 策略检查入口
```

审批决策流程:
```
ExecPolicyEngine::check(tool, params)
  → 匹配规则集
  → 评估风险等级
  → AskForApproval::OnRequest / OnModify / OnCreate / OnDelete
  → 用户决策 → Auto/Suggest/Never
```

### 第二层: Landlock 文件系统沙箱

```
Linux 5.13+ 内核特性
├── 读控制: 允许列表 (workspace, /tmp, /usr)
├── 写控制: 仅 workspace + /tmp
├── 执行控制: 禁止执行非白名单二进制
└── 网络控制: netns 命名空间隔离
```

Landlock 规则集 (`sandbox/landlock.rs`):
```rust
// 示例: 限制文件系统访问
let ruleset = LandlockRuleset::new()
    .allow_read("/usr")           // 允许读系统库
    .allow_read_write("/tmp")     // 允许临时文件
    .allow_read_write(workspace)  // 允许工作区
    .deny_all_else();             // 拒绝其他所有
```

### 第三层: 资源限制 (rlimits)

```
RLIMIT_CPU    — CPU 时间限制
RLIMIT_AS     — 内存地址空间限制
RLIMIT_FSIZE  — 文件大小限制
RLIMIT_NPROC  — 进程数限制
RLIMIT_NOFILE — 文件描述符限制
```

## 跨平台沙箱

| 平台 | 机制 | 文件 |
|------|------|------|
| Linux | Landlock + netns + rlimits | sandbox/landlock.rs, sandbox/backend.rs |
| macOS | Seatbelt (sandbox-exec) | sandbox/seatbelt.rs |
| Windows | Windows Sandbox | sandbox/windows.rs |
| 通用 | OpenSandbox 抽象层 | sandbox/opensandbox.rs, sandbox/mod.rs |

## 网络访问控制

```
NetworkPolicyDecider (network_policy.rs):
├── 白名单: IP/CIDR 范围
├── 白名单: 域名通配符 (*.ctfhub.com)
├── 会话级审批: /network allow <host>
├── 默认策略: 拒绝所有外部连接
└── Audit 日志: 所有网络请求记录

授权配置: ~/.secagent/authorization.toml
```

## 关键文件

- `crates/tui/src/sandbox/landlock.rs` — Landlock 规则引擎
- `crates/tui/src/sandbox/backend.rs` — 跨平台沙箱后端
- `crates/tui/src/sandbox/seatbelt.rs` — macOS Seatbelt
- `crates/tui/src/sandbox/opensandbox.rs` — OpenSandbox 抽象
- `crates/tui/src/sandbox/mod.rs` — 沙箱模块入口
- `crates/execpolicy/` — 执行策略引擎 (7 文件)
- `crates/tui/src/network_policy.rs` — 网络访问控制
- `~/.secagent/authorization.toml` — 授权白名单配置

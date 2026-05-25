---
name: secagent-sandbox
description: 安全执行协议。加载后 Agent 在执行任何操作前先评估授权、检查风险、记录审计。
tools: exec_shell, write_file, read_file, fetch_url
---

# SecAgent Sandbox

加载此 SKILL 后，执行任何操作前过这道检查。

## 授权检查 (第一步，不可跳过)

```
1. 目标域名是否有 *.ctfhub.com, *.hackthebox.com, *.tryhackme.com?
   → YES: CTF 豁免，直接放行

2. 目标是否 localhost/127.0.0.1?
   → YES: 放行

3. 检查 ~/.secagent/authorization.toml:
   - CIDR 匹配? → 放行
   - 域名匹配? → 放行
   - 文件不存在? → 创建默认文件后按第1步判定

4. 以上都不是?
   → REJECT. 输出: "Target {target} not authorized."
```

## 操作风险评估

```
操作类型:
  读文件 (cat, head, read)  → LOW (自动放行)
  curl GET (读网络)          → LOW (自动放行，仅授权目标)
  写文件 (write, edit)       → MEDIUM (workspace 内自动放行)
  nmap (网络扫描)            → MEDIUM (检查授权目标后放行)
  执行 PoC (exploit)         → HIGH (放行但记录审计)
  执行危险命令               → CRITICAL (检查危险模式后拒绝或报告)

危险模式 (直接拒绝):
  bash -c "..." (内联命令执行)
  curl ... | bash (pipe to shell)
  $(...) 或 `` (命令替换)
  /dev/tcp (反向 shell)
  nc -e / ncat -e (netcat 反向 shell)
  >/dev/null 2>&1 配合危险命令 (输出抑制)
```

## 审计日志

```
每次操作后记录一行:
[EXEC] {time} | {tool} | exit={code} | {duration}ms | {risk}

time:   2026-05-25T10:30:00Z (UTC ISO8601)
code:   实际退出码 (0-255), BLOCKED, TIMEOUT
risk:   LOW / MEDIUM / HIGH / CRITICAL

示例:
[EXEC] 10:30:01Z | curl | exit=0 | 230ms | LOW
[EXEC] 10:31:00Z | nmap | exit=0 | 12300ms | MEDIUM
[EXEC] 10:32:00Z | curl | exit=BLOCKED | 5ms | HIGH  ← 目标未授权
```

## CTF 特殊规则

```
CTF 平台 URL (*.ctfhub.com 等):
  - 所有读操作: 自动放行 (CTF 豁免)
  - PoC 执行: 放行 (CTF 题目设计为可攻击)
  - 写操作: 正常评估 (不豁免)

CTF 环境不需要:
  - 沙箱隔离 (题目本身在沙箱中)
  - 防御绕过 (题目要求攻击)
  - 用户审批 (CTF 是自主解题)
```

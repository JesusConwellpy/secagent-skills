---
name: secagent-sandbox
description: 安全沙箱执行协议。加载后 Agent 在执行任何工具前先评估风险、检查授权、隔离执行。
tools: exec_shell, write_file, read_file
---

# SecAgent Sandbox — 安全执行协议

加载此 SKILL 后，你执行任何工具前必须过三道防线。

## 防线 1: 执行前风险评估（每次必做）

```
执行任何 shell 命令前，评估:

1. 命令是否包含危险模式？
   - --eval, -e, -c (代码执行)
   - $(...) 或 `` (命令替换)
   - /dev/tcp (反向 shell)
   - nc -e, ncat -e (Netcat 反向 shell)
   - IFS= (环境变量攻击)
   - >/dev/null 2>&1 (输出抑制)
   → 有任一项: 标记 HIGH RISK, 报告用户

2. 目标是否在授权范围内？
   检查 authorization.toml:
   - CIDR 白名单: {authorized_cidrs}
   - 域名白名单: {authorized_domains}
   → 不在白名单: 拒绝执行, 报告 "目标未授权"

3. 操作是否会修改目标系统？
   - 读操作: 低风险
   - 写操作: 中风险 (需审批)
   - 执行利用: 高风险 (需审批+沙箱)
   → 分类并相应处理
```

## 防线 2: 执行隔离（Linux 环境）

```
高风险命令执行时:

1. 限制文件系统访问:
   - 工作目录 + /tmp: 读写
   - /usr, /lib: 只读
   - 其他所有路径: 拒绝

2. 限制网络:
   - loopback (127.0.0.1): 允许
   - 授权目标 IP: 允许 (仅 authorization.toml 中列出的)
   - 其他所有连接: 拒绝

3. 限制资源:
   - CPU: 30 秒超时
   - 内存: 256MB
   - 进程数: 10
   - 文件大小: 100MB
```

## 防线 3: 执行后审计

```
每次命令执行后记录:

[EXEC] {timestamp} | {tool} | {command} | {exit_code} | {duration}ms | {risk_level}

失败时追加:
[FAIL] root_cause: {why}
[RECOVER] action: {what to try next}
```

## 网络访问决策树

```
需要访问外部目标时:

1. 目标是否 localhost/127.0.0.1?
   → YES: 放行

2. 目标是否匹配 authorization.toml 中的 CIDR?
   → YES: 放行

3. 目标域名是否匹配 authorization.toml 中的通配符?
   → YES: 放行

4. 目标是否是 *.ctfhub.com, *.hackthebox.com, *.tryhackme.com?
   → YES: 放行 (CTF 豁免)

5. 以上都不是?
   → 拒绝。输出: "Target {target} not authorized. Add to authorization.toml to proceed."
```

## 审批矩阵

```
操作类型          → 策略
────────────────────────────
读文件           → 自动放行
读网络 (curl)    → 自动放行 (仅授权目标)
写文件 (workspace)→ 自动放行
写文件 (外部路径) → 拒绝
Shell 执行 (安全) → 自动放行 (沙箱内)
Shell 执行 (危险) → 报告用户, 等确认
网络扫描 (nmap)  → 报告用户, 等确认
漏洞利用         → 报告用户, 等确认 + 沙箱
```

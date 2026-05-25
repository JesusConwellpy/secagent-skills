---
name: secagent-methodology
description: 安全攻击方法论。加载后获得 4 阶段渗透测试循环、CVE 匹配策略、防御绕过技术选择、观察→假设→验证→结论的完整证据链。
tools: agent_spawn, web_search, fetch_url, exec_shell, read_file, write_file
---

# SecAgent Methodology — 攻击方法论

加载此 SKILL 后，你获得完整的渗透测试方法论。按以下流程执行，不可跳过。

## 4 阶段循环（每次 Engagement 必须完整走完）

```
Phase 1: RECONNAISSANCE (侦察)
  → Phase 2: ANALYSIS (分析)
    → Phase 3: EXPLOITATION (利用)
      → Phase 4: REPORTING (报告)
```

## Phase 1: RECONNAISSANCE — 广泛观察

```
目标: 映射攻击面。不遗漏任何服务。

步骤:
1. spawn type="recon" prompt="枚举 {target}: 端口、子域名、服务版本、OSINT"
   多目标 → 每个目标一个 recon agent，同回合并行发出

2. 如果有 Web 服务 → spawn type="web-analyst"
   prompt="映射 {target} 的 Web 攻击面: 端点、参数、headers、表单"

3. 如果有二进制文件 → spawn type="binary-analyst"
   prompt="分析 {binary}: checksec、strings、disassemble entry point"

4. wait(all agents) → 收集所有结果

原则: 多扫一个端口只要几秒。漏一个服务毁整个 engagement。
```

## Phase 2: ANALYSIS — 形成假设

```
步骤:
1. 评估每个发现:
   Port open + service unknown    → LOW    (需要指纹)
   Service + version confirmed    → MEDIUM (需要 CVE 匹配)
   CVE matched + PoC available    → HIGH   (需要验证)
   PoC executed successfully      → CONFIRMED (立即报告)

2. CVE 匹配 (优先级递降):
   a. 搜 "{product} {version} CVE" → NVD, exploit-db, GitHub
   b. 搜 "{product} CVE" (邻近版本 ±2 minor)
   c. 搜 "{vuln_type} {product}" (同类漏洞)
   d. 以上都找不到 → 报告用户, 建议扩大范围

3. 每个 HIGH 置信度发现 → 形成可测试假设:
   HYPOTHESIZE: "{vuln} affects {target} because {reason}"
   PREDICT: "If correct, {payload} will produce {expected_result}"

4. 按优先级排序: 确认度 × 影响度
```

## Phase 3: EXPLOITATION — 系统验证

```
铁律: 一个假设 = 一个 agent。不批量。

步骤:
1. 对每个 HIGH 置信度假设:
   spawn type="exploit-runner" prompt="验证 {CVE-ID} on {target}. Expected: {expected}"

2. 结果处理:
   PoC 成功 → CONFIRMED。记录证据。推进计划。
   PoC 失败 → 分析三步:
     a. 假设错了? → 新假设, 新 agent
     b. 测试错了? → 修正, 重试
     c. 补丁已应用? → 记录排除, 下一个

3. 防御绕过 (选择正确技术):
   遇到 NX/DEP     → ROP / ret2libc
   遇到 ASLR       → Info leak / partial overwrite
   遇到 Canary     → Format string leak
   遇到 PIE        → Partial overwrite (12-bit)
   遇到 Full RELRO → __malloc_hook / exit hooks
   遇到 WAF        → 编码变异 / HPP / HTTP smuggling
   遇到 CSP        → JSONP / AngularJS CSTI / DOM clobbering
   遇到 Input Val  → Unicode normalize / double-encode / null-byte
```

## Phase 4: REPORTING — 证据合成

```
步骤:
1. 排序: Critical > High > Medium > Low > Info
2. 每个 Finding 用此模板:

## F{###}: {title} | Severity: {level}
**Target**: [REDACTED] | **Type**: {sqli/xss/rce/bof}
**CVSS**: {score} | **Confirmed**: Yes/No
**Description**: {one paragraph}
**Impact**: {concrete impact}
**Reproduction**: {step by step}
**Remediation**: {specific fix}

3. 写入 cases/{case-name}/report.md
4. 更新 Wiki (加载了 secagent-knowledge 时)
```

## 观察→动作 速查表

```
观察到                               → 动作
─────────────────────────────────────────────────────────────
Port {N} open, service unknown       → spawn recon (fingerprint service)
{Product} {version} on port {N}     → spawn intel-gatherer (CVE search)
SQL error on '                       → spawn web-analyst (SQLi verify)
<script> executed in response        → spawn web-analyst (XSS verify)
../../../etc/passwd works            → spawn web-analyst (LFI verify)
; id returns uid=0                   → spawn exploit-runner (cmd inject)
EIP = 0x41414141                     → spawn binary-analyst (BOF verify)
sudo -l shows NOPASSWD               → spawn exploit-runner (PrivEsc)
Binary file provided                 → spawn binary-analyst (analyze)
3+ targets in scope                  → spawn recon × N (parallel)
CVE matched + PoC available          → spawn exploit-runner (verify)
```

## 协调自检清单

```
每次 spawn 前:    这是并行任务吗? → 同回合发出
每次 wait 后:     所有结果都评估了吗? → 形成假设
每次 PoC 后:      验证结果记录了吗? → 写 Wiki
Phase 结束:       下一阶段可以开始吗? → 证据够了吗?
Engagement 结束:  报告写了吗? → 所有 Finding 有模板
```

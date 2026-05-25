---
name: secagent-methodology
description: 安全攻击方法论 — 4 阶段渗透测试循环 (Recon/Analysis/Exploitation/Reporting)、CVE 渐进匹配、防御绕过知识库、攻击面自动映射
version: "1.0"
source: JesusConwellpy/SecAgent-TUI (crates/tui/src/prompts/base.md, crates/tui/assets/personas/)
tools: agent_spawn, web_search, fetch_url, read_file, write_file
---

# SecAgent Methodology — 攻击方法论与推理链

## 能力概述

注入完整渗透测试方法论。4 阶段安全研究循环、CVE 渐进匹配策略、二进制+Web 双域防御绕过技术库、观察→假设自动映射表。每步形成完整证据链。

## 调用链 (完整端到端)

```
安全目标输入 (target + task)
  │
  ├─→ Phase 1: RECONNAISSANCE (侦察 — 广泛观察)
  │   │
  │   ├─ 并行 Fan-Out:
  │   │   spawn recon × N (每个目标一个)
  │   │   spawn web-analyst (Web 攻击面)
  │   │   spawn binary-analyst (如果有二进制文件)
  │   │
  │   ├─ 攻击面映射:
  │   │   端口 → 服务 → 版本 → 已知漏洞
  │   │   子域名 → Web 端点 → 参数 → 注入点
  │   │   二进制 → checksec → 反汇编 → 漏洞模式
  │   │
  │   └─ 成本不对称原则:
  │       额外端口扫描: 秒级成本
  │       遗漏服务: 整个 engagement 失败
  │       → Scan broadly. Verify narrowly.
  │
  ├─→ Phase 2: ANALYSIS (分析 — 形成假设)
  │   │
  │   ├─ 证据评估:
  │   │   每条发现 → 置信度评分 (LOW/MEDIUM/HIGH/CONFIRMED)
  │   │   可测试、可证伪的假设
  │   │   优先级 = 确认度 × 影响度
  │   │
  │   ├─ CVE 渐进匹配:
  │   │   Level 1: 精确 product+version → NVD CPE, exploit-db, GitHub
  │   │   Level 2: 邻近版本 (±2 minor)
  │   │   Level 3: 同漏洞类型 (不同产品)
  │   │   Level 4: 升级用户, 建议扩大范围
  │   │
  │   └─ 假设形成模板:
  │       OBSERVE: "Port 3306 open. MySQL 5.7.38."
  │       HYPOTHESIZE: "May be vulnerable to CVE-2021-XXXX (auth bypass)"
  │       PREDICT: "If correct, PoC will grant root shell."
  │
  ├─→ Phase 3: EXPLOITATION (利用 — 系统验证)
  │   │
  │   ├─ 铁律:
  │   │   ONE hypothesis per agent. Never batch verification.
  │   │   批量 = 不知道哪个假设是错的.
  │   │
  │   ├─ 验证协议:
  │   │   1. spawn exploit-runner with EXACT CVE ID + target
  │   │   2. Agent returns: success/failure + evidence
  │   │   3. Success → CONFIRMED, advance plan
  │   │   4. Failure → 分析原因:
  │   │      - 假设错了? → form new hypothesis
  │   │      - 测试错了? → correct test, retry
  │   │      - 补丁已应用? → document exclusion
  │   │
  │   └─ 防御绕过:
  │       二进制: NX→ROP, ASLR→InfoLeak, Canary→Leak, PIE→PartialOverwrite
  │       Web:    WAF→EncodingVar, CSP→JSONP, Validation→UnicodeNorm
  │
  └─→ Phase 4: REPORTING (报告 — 证据合成)
      │
      ├─ 严重性排序: Critical > High > Medium > Low > Info
      ├─ 结构化报告:
      │   ## F###: <title> | Severity: Critical/High/Medium/Low
      │   Target: [REDACTED] | Type: <SQLi/XSS/RCE/BoF>
      │   CVSS: X.X | Confirmed: Yes/No
      │   Description: <one paragraph>
      │   Impact: <concrete impact>
      │   Reproduction: <step-by-step>
      │   Remediation: <specific fix>
      │
      └─ 证据存储: cases/<case>/F###/finding.md
```

## 证据评估阈值 (完整)

```
LOW confidence:
  条件: 开放端口 (未指纹)
  动作: 需要服务指纹识别
  示例: "Port 80 open" → need service scan

MEDIUM confidence:
  条件: 服务版本确认
  动作: 需要 CVE 匹配
  示例: "Apache 2.4.51 detected" → search CVEs

HIGH confidence:
  条件: CVE 匹配 + PoC 可用
  动作: 需要验证
  示例: "CVE-2023-XXXX matched, PoC at exploit-db"

CONFIRMED:
  条件: PoC 执行成功
  动作: 立即报告
  示例: "PoC successful, root shell obtained"
```

## CVE 渐进匹配策略

```rust
// 伪代码 — 匹配优先级
fn find_cve(product: &str, version: &str) -> Vec<CveMatch> {
    // Level 1: Exact match
    let exact = search_nvd(product, version);
    if !exact.is_empty() { return exact; }

    // Level 2: Adjacent versions
    let adjacent = search_nvd_range(product, version, 2);
    if !adjacent.is_empty() { return adjacent; }

    // Level 3: Same vulnerability class
    let vuln_class = classify_vuln_type(product);
    let similar = search_by_class(vuln_class);

    // Level 4: Escalate
    escalate_to_user(product, version);
    vec![]
}
```

## 观察→子代理映射表 (完整)

```
| 观察 (atomic fact)            | 漏洞类型         | Spawn           |
|------------------------------|-----------------|-----------------|
| SIGSEGV at 0x41414141        | Buffer Overflow | binary-analyst  |
| EIP = 0x41414141             | BOF confirmed   | exploit-runner  |
| SQL error on '               | SQL Injection   | web-analyst     |
| SLEEP(5) → 5.1s delay        | Time-based SQLi | web-analyst     |
| User input echoed in page    | XSS             | web-analyst     |
| <script>alert(1)</script>    | XSS confirmed   | exploit-runner  |
| ; id returns uid=0           | Cmd Injection   | exploit-runner  |
| ../../../etc/passwd works    | File Inclusion  | web-analyst     |
| URL param fetches internal   | SSRF            | web-analyst     |
| 192.0.2.169 accessible       | SSRF→cloud meta | web-analyst     |
| Base64→rO0AB or O:N:         | Deserialization | exploit-runner  |
| sudo -l NOPASSWD / SUID      | PrivEsc         | exploit-runner  |
| NX ON, no PIE, partial RELRO | ROP needed      | binary-analyst  |
| Product + version identified | CVE match       | intel-gatherer  |
```

## 二进制防御绕过 (完整知识库)

```
| 缓解措施     | 绕过技术                              | 前置条件                    |
|-------------|--------------------------------------|---------------------------|
| NX/DEP      | ROP (Return-Oriented Programming)    | Gadget chain + stack ctrl |
| NX/DEP      | ret2libc                             | libc base leak            |
| NX/DEP      | JOP (Jump-Oriented Programming)      | Jump gadget chain         |
| ASLR        | Info leak (memory disclosure)        | Read primitive            |
| ASLR        | Partial overwrite (2 bytes)          | Write primitive           |
| ASLR        | Heap spray + predict                 | Large allocation quota    |
| Canary      | Format string leak                   | Format string vuln        |
| Canary      | __stack_chk_fail overwrite           | Write-what-where          |
| Canary      | Brute force (forking server)         | Fork server + ASLR off    |
| PIE         | Partial overwrite (12 bits static)   | Stack buffer overflow     |
| PIE         | Info leak                            | Memory disclosure         |
| Full RELRO  | __malloc_hook / __free_hook          | Heap control + libc leak  |
| Full RELRO  | Exit hooks (atexit destructors)      | Write primitive           |
| Shadow Stack| Signal handler corruption            | Signal handler access     |
| CFI         | COOP (Counterfeit Object-Oriented)   | Virtual call gadgets      |
```

## Web 防御绕过 (完整知识库)

```
| 防御       | 绕过技术                                    | 原理                       |
|-----------|-------------------------------------------|---------------------------|
| WAF       | URL 编码变异 (%u0027, %2527, double-encode)| Parser differential        |
| WAF       | HTTP Parameter Pollution (HPP)            | 参数处理歧义               |
| WAF       | HTTP Request Smuggling (CL.TE / TE.CL)    | 前端/后端解析不一致         |
| WAF       | 大小写变异 (SeLeCt / sElEcT)               | 正则大小写不敏感           |
| WAF       | 内联注释 (SEL/**/ECT)                      | Tokenizer bypass           |
| CSP       | JSONP 端点 (callback=<script>...</script>) | 同源策略例外               |
| CSP       | AngularJS CSTI ({{constructor.constructor}})| Template injection        |
| CSP       | DOM clobbering                            | JS 变量覆盖                |
| CSP       | Script gadgets (existing JS on page)      | 利用页面已有 JS            |
| Validation| Unicode normalize (NFD/NFC)               | 规范化后字符变化           |
| Validation| Null-byte injection (%00)                 | 字符串截断                 |
| Validation| Parser differential (Content-Type 混淆)   | 不同解析器不同结果          |
| Rate Lim  | IP rotation (X-Forwarded-For spoofing)    | 身份伪造                   |
| CSRF      | XSS → token extraction                    | 跨域读取                   |
```

## 协调反模式 (永远不要做)

```
1. 直接执行: "Let me run nmap myself"
   → 错误。spawn recon agent.

2. 批量验证: "Test SQLi AND XSS in one agent"
   → 错误。ONE hypothesis per agent.

3. 静默失败: Agent 失败了但你不分析原因
   → 错误。Form new hypothesis.

4. 过早结论: "Port 80 is open, probably Apache"
   → 错误。等指纹结果。

5. 串行执行: Recon THEN web-analyst sequentially
   → 错误。Spawn both in parallel.
```

## 关键文件 (源仓库)

| 文件 | 行数 | 用途 |
|------|------|------|
| `crates/tui/src/prompts/base.md` | 120 | 4 阶段方法论 (Security 部分) |
| `crates/tui/assets/personas/recon.md` | 50+ | 侦察 Agent |
| `crates/tui/assets/personas/web-analyst.md` | 50+ | Web 分析 Agent |
| `crates/tui/assets/personas/binary-analyst.md` | 50+ | 二进制分析 Agent |
| `crates/tui/assets/personas/exploit-runner.md` | 50+ | 漏洞利用 Agent |
| `crates/tui/assets/personas/intel-gatherer.md` | 50+ | 情报收集 Agent |
| `crates/tui/assets/personas/code-auditor.md` | 50+ | 代码审计 Agent |
| `crates/tui/assets/personas/post-exploit.md` | 50+ | 后渗透 Agent |
| `crates/tui/src/tools/plan.rs` | 400+ | PlanState (攻击计划) |

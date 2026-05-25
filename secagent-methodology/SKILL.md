---
name: secagent-methodology
description: 安全攻击方法论 — 4 阶段渗透测试循环 (Recon/Analysis/Exploitation/Reporting)、CVE 渐进匹配、防御绕过知识库、攻击面映射
---

# SecAgent Methodology — 攻击方法论与推理链

## 能力概述

注入完整的渗透测试方法论。包含 4 阶段安全研究循环、CVE 渐进匹配策略、防御绕过技术表、攻击面自动映射。每条观察→假设→验证→结论形成完整的证据链。

## 调用链

```
安全目标输入
  → Phase 1: RECONNAISSANCE
    → 并行 spawn: recon × N + web-analyst + binary-analyst
    → 攻击面映射: 端口/服务/子域名/端点/参数
    → 信息收集: WHOIS/DNS/OSINT
  → Phase 2: ANALYSIS
    → CVE 渐进匹配:
        1. 精确匹配 (product + version) → NVD, exploit-db, GitHub
        2. 邻近版本 (±2 minor)
        3. 同类型漏洞 (产品不匹配 → 漏洞类型匹配)
        4. 升级给用户
    → 假设形成: 可测试、可证伪
    → 优先级排序: 确认度 × 影响度
  → Phase 3: EXPLOITATION
    → 每个假设一个 exploit-runner (不批量)
    → 防御绕过: NX→ROP, ASLR→Info leak, Canary→Leak
    → PoC 执行: 沙箱隔离
    → 失败分析: 假设错了？测试错了？
  → Phase 4: REPORTING
    → 严重性排序: Critical > High > Medium > Low
    → 结构化报告: description, impact, PoC, remediation
    → 写入 case 文件: cases/<case>/report.md
```

## 证据评估阈值

| 证据等级 | 条件 | 置信度 |
|---------|------|--------|
| LOW | 开放端口 (未指纹) | 需进一步侦察 |
| MEDIUM | 服务版本确认 | 需 CVE 匹配 |
| HIGH | CVE 已匹配 + PoC 可用 | 需验证 |
| CONFIRMED | PoC 执行成功 | 立即报告 |

## CVE 渐进匹配策略

```
1. 精确匹配: product + version → NVD CPE match
   ↓ 未命中
2. 邻近版本: same product, ±2 minor versions
   ↓ 未命中
3. 同类型漏洞: no product match → vulnerability class in similar products
   ↓ 未命中
4. 升级: escalate to user, suggest scope expansion
```

## 防御绕过知识库

### 二进制防御

| 缓解措施 | 绕过技术 |
|---------|---------|
| NX/DEP | ROP, ret2libc, JOP |
| ASLR | Info leak, partial overwrite |
| Canary | Format string leak, __stack_chk_fail overwrite |
| PIE | 12-bit static partial overwrite, info leak |
| Full RELRO | __malloc_hook, __free_hook, exit hooks |

### Web 防御

| 防御 | 绕过 |
|------|------|
| WAF | 编码变异, HPP, HTTP smuggling |
| CSP | JSONP, AngularJS CSTI, script gadgets |
| Input validation | Unicode normalize, double-encode, null-byte |

## 观察→子代理映射表

| 观察 | 漏洞类型 | 应 spawn |
|------|---------|---------|
| `SIGSEGV at 0x41414141` | Buffer Overflow | `binary-analyst` |
| `SQL error on '` | SQL Injection | `web-analyst` |
| `User input echoed in page` | XSS | `web-analyst` |
| `` `; id` returns uid=0 `` | Command Injection | `exploit-runner` |
| `../../../etc/passwd works` | File Inclusion | `web-analyst` |
| `URL param fetches internal IPs` | SSRF | `web-analyst` |
| `Base64→rO0AB` | Deserialization | `exploit-runner` |
| `sudo -l NOPASSWD` | PrivEsc | `exploit-runner` |
| `Product + version identified` | CVE match | `intel-gatherer` |

## 关键文件

- `crates/tui/src/prompts/base.md` — 4 阶段方法论 (Security 模式部分)
- `crates/tui/assets/personas/recon.md` — 侦察 Agent Persona
- `crates/tui/assets/personas/web-analyst.md` — Web 分析 Agent Persona
- `crates/tui/assets/personas/exploit-runner.md` — 漏洞利用 Agent Persona
- `crates/tui/assets/personas/intel-gatherer.md` — 情报收集 Agent Persona
- `crates/tui/src/prompts/approvals/` — 审批策略 (Auto/Suggest/Never)

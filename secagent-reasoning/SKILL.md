---
name: secagent-reasoning
description: 侦探推理引擎。加载后获得线索提取、因果图构建、异常检测、失败模式匹配、攻击链识别能力。
tools: read_file, write_file, grep_files
---

# SecAgent Reasoning — 侦探推理引擎

加载此 SKILL 后，你获得结构化安全推理能力。每次分析必须建立证据链。

## 推理工作流（每次分析安全数据时执行）

```
输入: 工具输出 / Agent 报告 / 扫描结果

Step 1: CLUE EXTRACTION (线索提取)
  → 从原始输出中提取所有线索

Step 2: CLUE CORRELATION (交叉关联)
  → 找出线索之间的关系

Step 3: ANOMALY DETECTION (异常检测)
  → 运行 12 个检测器

Step 4: HYPOTHESIS FORMATION (假设形成)
  → 每个异常形成一个可测试假设

Step 5: EVIDENCE CHAIN VALIDATION (证据链校验)
  → 检查完整性、一致性、可靠性

输出: 结构化推理结果
```

## Step 1: 线索提取

```
从任意文本中提取以下类型的线索:

正则模式:
  IP:        \b(?:\d{1,3}\.){3}\d{1,3}\b
  Domain:    \b[a-zA-Z0-9-]+(?:\.[a-zA-Z]{2,})+\b
  CVE:       CVE-\d{4}-\d{4,}
  Port:      端口: (\d{2,5})
  Version:   (\d+\.\d+(?:\.\d+)?)
  Email:     \b[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}\b
  URL:       https?://[^\s]+

语义模式:
  Vuln type:   SQLi / XSS / RCE / BoF / LFI / SSRF / CSRF / PrivEsc / ...
  Technique:   ROP / ret2libc / heap spray / format string / ...
  Tool:        nmap / burp / metasploit / sqlmap / gdb / ...

输出格式:
  CLUE-{id}: {type}={value} | source={tool/agent} | confidence={0.0-1.0}
  例: CLUE-001: Port=3306 | source=nmap | confidence=1.0
      CLUE-002: Service=MySQL | source=nmap | confidence=0.9
      CLUE-003: Version=5.7.38 | source=nmap | confidence=0.8
```

## Step 2: 交叉关联

```
关联规则:

1. 空间关联: 同一 IP/主机上的线索
   CLUE-001(192.168.1.1:80) + CLUE-002(192.168.1.1:443)
   → HOST-001: 192.168.1.1 running HTTP+HTTPS

2. 时间关联: 同一时间段内的事件
   适合追踪攻击链的时序

3. 因果关联: A 导致 B
   Port open → Service running → Version fingerprinted → CVE matched
   这是核心推理链

4. 类型关联: 同一漏洞类型的不同实例
   多个 SQLi 发现 → 可能存在系统性的输入验证问题

输出格式:
  REL-{id}: {clue_a} --[{relation}]--> {clue_b}
  例: REL-001: CLUE-001 --[port_of]--> CLUE-002
      REL-002: CLUE-002 --[has_version]--> CLUE-003
```

## Step 3: 异常检测（12 个检测器）

```
对每一组线索运行以下检测器:

1. PortAnomaly:
   端口是否在标准服务端口之外? 4444, 31337, 1337?
   → ANOMALY: "Non-standard port {port} ({service})"

2. ServiceAnomaly:
   服务版本是否已知存在漏洞?
   → ANOMALY: "{service} {version} has {n} known CVEs"

3. CVEAnomaly:
   精确 product+version 是否匹配 CVE?
   → ANOMALY: "CVE-{id} matches {product} {version}"

4. CredentialAnomaly:
   是否发现默认凭证或弱密码?
   → ANOMALY: "Default credentials: {user}/{pass}"

5. ConfigAnomaly:
   是否存在不安全配置?
   → ANOMALY: "Directory listing enabled on {path}"

6. PatchAnomaly:
   已知漏洞是否有补丁?
   → ANOMALY: "{CVE} unpatched — fixed in {version}"

7. ChainAnomaly:
   多条线索是否形成攻击链?
   检查: Recon 线索 → Vuln 线索 → Exploit 线索 → PostExploit 线索
   → ANOMALY: "Attack chain detected: {steps}" SEVERITY: CRITICAL

8-12. TrafficAnomaly, FileAnomaly, ProcessAnomaly, LogAnomaly, BehaviorAnomaly:
   有相关数据时触发，否则跳过
```

## Step 4: 假设形成

```
每个异常 → 一个可测试假设

模板:
  H-{id}: {claim}
  Based on: [{clue_ids}]
  Test: {specific test action}
  Expected if true: {what to observe}
  Expected if false: {what to observe}
  Confidence: {0.0-1.0}

示例:
  H-001: CVE-2021-XXXX auth bypass is exploitable on target
  Based on: [CLUE-003(MySQL 5.7.38), CLUE-005(CVE-2021-XXXX matched)]
  Test: Execute CVE-2021-XXXX PoC against 192.168.1.1:3306
  Expected if true: Root shell obtained
  Expected if false: Connection refused or access denied
  Confidence: 0.7
```

## Step 5: 证据链校验

```
对每条结论检查:

1. 完整性:
   □ 每个 Hypothesis 是否有 Observation 支撑?
   □ 每个 Conclusion 是否有 Verification 支撑?

2. 一致性:
   □ 是否有矛盾的证据?
   □ 两个 Hypothesis 是否互斥?

3. 可靠性:
   □ 证据来源是否可信? (工具扫描 > Agent 推断 > 外部信息)
   □ 是否有独立验证?

4. 可重复性:
   □ 验证步骤是否明确?
   □ 不同环境是否能复现?

输出:
  [PASS] 或 [FAIL: {reason}]
  置信度: {0.0-1.0}
```

## 失败模式匹配

```
验证失败时:

1. 匹配已知失败模式:
   - "Connection refused" → 目标不可达或服务关闭
   - "Access denied" → 补丁已应用或凭据错误
   - "Timeout" → 防火墙或网络隔离
   - "Unexpected output format" → 工具版本不兼容

2. 选择恢复路径:
   Connection refused → 检查目标可达性 → retry
   Access denied → 检查补丁版本 → 寻找替代 CVE
   Timeout → 检查防火墙规则 → 更换端口/协议
   Unexpected output → 更换工具版本 → 手动验证

3. 记录失败:
   写入 failures.log:
   [FAIL] {timestamp} | {tool} | {error} | {root_cause} | {recovery_attempted} | {success}
```

## 推理输出模板

```
═══════════════════════════════════
REASONING REPORT
═══════════════════════════════════
Input: {source}
Clues extracted: {n}
  CLUE-001: {type}={value} | {source} | confidence={c}
  ...

Correlations: {m}
  REL-001: {a} --[{rel}]--> {b}
  ...

Anomalies: {k}
  ANOM-001: [{detector}] {description} | SEVERITY: {level}
  ...

Hypotheses: {h}
  H-001: {claim} | Test: {test} | Confidence: {c}
  ...

Evidence Chain: {PASS or FAIL: reason} | Overall confidence: {c}
═══════════════════════════════════
```

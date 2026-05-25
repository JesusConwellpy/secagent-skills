---
name: secagent-knowledge
description: 本地 LLM-Wiki 知识系统。加载后 Agent 在本地创建结构化 Wiki、FTS 搜索、跨 Agent 共享、跨会话继承。核心：知识生产→消化→检索→复用的完整闭环。
tools: read_file, write_file, list_dir, grep_files
---

# SecAgent Knowledge — 本地 LLM-Wiki 知识系统

加载此 SKILL 后，你获得完整的本地知识管理能力。这不是"读文档"——你实际创建和维护一个结构化的本地 Wiki。

## 自举：首次加载时立即执行

```
1. 确定 Wiki 根目录:
   WIKI_ROOT = {workspace}/wiki

2. 创建目录骨架:
   mkdir -p {WIKI_ROOT}/entities/ips
   mkdir -p {WIKI_ROOT}/entities/domains
   mkdir -p {WIKI_ROOT}/entities/cves
   mkdir -p {WIKI_ROOT}/entities/tools
   mkdir -p {WIKI_ROOT}/concepts/attack-methods
   mkdir -p {WIKI_ROOT}/concepts/defense-tech
   mkdir -p {WIKI_ROOT}/concepts/design-patterns
   mkdir -p {WIKI_ROOT}/discoveries/scan-results
   mkdir -p {WIKI_ROOT}/discoveries/vuln-findings
   mkdir -p {WIKI_ROOT}/discoveries/pocs
   mkdir -p {WIKI_ROOT}/synthesis/reports
   mkdir -p {WIKI_ROOT}/synthesis/analysis
   mkdir -p {WIKI_ROOT}/synthesis/summaries

3. 创建索引文件:
   写入 {WIKI_ROOT}/INDEX.md，内容:
   "# Wiki Index\n\nLast updated: {timestamp}\n\n## Entities\n\n## Discoveries\n\n## Synthesis\n"

4. 确认: "Wiki ready at {WIKI_ROOT}。现在开始生产知识。"
```

## 知识生产→消化→检索→复用 闭环

### 环节 1: 生产 (每当有安全发现时)

你做任何操作后，**立即**将发现写入 Wiki。不要等、不要攒。

```
触发条件 → 动作
─────────────────────────────────────────────
扫描发现 IP+端口+服务      → 写 entities/ips/{ip}.md
识别到产品+版本            → 写 entities/cves/{cve-id}.md (或创建占位)
CVE 匹配成功               → 写 entities/cves/{cve-id}.md
漏洞验证确认               → 写 discoveries/vuln-findings/F{###}.md
PoC 执行成功               → 写 discoveries/pocs/{finding-id}-poc.md
子代理返回结果             → 提取关键发现 → 写入对应目录
攻击方法/模式归纳          → 写 concepts/attack-methods/{technique}.md
防御绕过成功               → 写 concepts/defense-tech/{bypass}.md
```

写入规则:
- 文件名 = 实体标识符 (IP、CVE 编号、Finding ID)
- 内容使用下面的精确模板
- 写完后追加一行到 INDEX.md 对应节

### 环节 2: 消化 (结构化写入)

**每个条目使用精确模板**。不要自由发挥——结构一致才能被搜索命中。

#### IP 条目模板

写入 `wiki/entities/ips/{ip}.md`:

```markdown
# {ip}

- **First seen**: {timestamp}
- **Last seen**: {timestamp}
- **Hostname**: {hostname or "unknown"}
- **OS**: {os guess or "unknown"}

## Open Ports
| Port | Service | Version | CVE |
|------|---------|---------|-----|
| {port} | {service} | {version} | {cve or "—"} |

## Findings
- [{finding-id}](../discoveries/vuln-findings/{finding-id}.md): {one-line summary}
```

#### CVE 条目模板

写入 `wiki/entities/cves/{cve-id}.md`:

```markdown
# {cve-id}

- **Product**: {product} {version}
- **Type**: {sqli/xss/rce/bof/lfi/ssrf/...}
- **CVSS**: {score}
- **CWE**: {cwe-id}
- **Exploit**: {link or "no public exploit"}
- **PoC**: [poc](../discoveries/pocs/{finding-id}-poc.md)
- **Fixed in**: {version or "unknown"}
- **Verified**: {yes/no/untested}
- **Verified against**: {target or "—"}
```

#### 漏洞发现条目模板

写入 `wiki/discoveries/vuln-findings/F{###}.md`:

```markdown
# F{###}: {title}

- **Target**: [REDACTED]
- **Endpoint**: {url or path}
- **Type**: {vuln type}
- **Severity**: Critical / High / Medium / Low
- **Status**: Confirmed / Pending / False Positive
- **CVSS**: {score or "—"}

## Evidence
{tool output, screenshots, payloads}

## Impact
{concrete impact — data exposed, access gained, system compromised}

## Reproduction
1. {step 1}
2. {step 2}
3. {expected result}

## Remediation
{specific fix — not "patch it", but exactly what to change}
```

#### 报告模板

写入 `wiki/synthesis/reports/{date}-{engagement}.md`:

```markdown
# {engagement name} — Security Assessment Report

**Date**: {date}
**Scope**: {targets}
**Findings**: {count}

## Severity Breakdown
- Critical: {n}
- High: {n}
- Medium: {n}
- Low: {n}

## Findings
{list of F### links with one-line summaries}

## Timeline
{chronological log of key actions and discoveries}

## Recommendations
{prioritized list of remediation steps}
```

### 环节 3: 检索 (开始新工作前必做)

开始任何安全任务前，**先搜 Wiki**。这是铁律。

```
搜索步骤:
1. grep_files("wiki/", "{keyword}")           — 文件名搜索
2. read_file("wiki/INDEX.md")                  — 查看索引
3. grep_files("wiki/", "{product} {version}")  — 精确产品搜索
4. grep_files("wiki/", "{cve-id}")             — CVE 搜索
5. read_file("wiki/entities/ips/{target}.md")  — 目标已有信息

搜索优先级:
1. 先搜 IP/域名 → entities/ips/ 或 entities/domains/
2. 再搜 CVE → entities/cves/
3. 再搜漏洞类型 → discoveries/vuln-findings/
4. 再搜攻击方法 → concepts/attack-methods/
5. 最后搜报告 → synthesis/
```

### 环节 4: 复用 (跨 Agent / 跨会话)

```
同会话内共享:
  Agent A 写入 → Agent B 立刻可读 (同文件系统)
  协议: Agent A write → Agent A 报告文件路径 → Agent B read

跨会话继承:
  新会话启动 → 搜索 wiki/INDEX.md
  → 读最近 3 天修改的条目
  → 注入上下文: "Previous session discovered: {summary}"
  → 继续工作, 不重复已完成的发现

策展 (每 10 次写入或会话结束):
  1. 检查 90 天未访问的条目 → 标记 [STALE]
  2. 合并重复条目 (同实体多条记录)
  3. 更新 INDEX.md
  4. 报告: "Wiki 策展完成: {n} stale, {m} merged, 总计 {total} 条目"
```

## 跨 Agent 知识共享协议

这是让多个 Agent 在同一个 Engagement 中协作的协议:

```
## Agent A (侦察) 产出知识
→ write "wiki/entities/ips/192.168.1.1.md"
→ write "wiki/discoveries/scan-results/nmap-192.168.1.1.md"

## Agent B (情报) 消费+生产
→ grep_files("wiki/", "192.168.1.1")  # 读 Agent A 的侦察结果
→ grep_files("wiki/", "CVE MySQL 5.7")  # 搜已有 CVE 知识
→ write "wiki/entities/cves/CVE-2021-XXXX.md"  # 生产新 CVE 条目

## Agent C (利用) 消费+生产
→ read_file("wiki/entities/cves/CVE-2021-XXXX.md")  # 读 Agent B 的 CVE 条目
→ write "wiki/discoveries/pocs/F001-poc.md"  # 生产 PoC 条目

## 协调员 消费
→ list_dir("wiki/")  # 查看全部产出
→ 合成 → write "wiki/synthesis/reports/2026-05-25-engagement.md"
```

## 知识质量自检

每次写入后自问:
- [ ] 文件名是否可搜索？(用 IP、CVE 编号、Finding ID，不要用描述性标题)
- [ ] 模板是否完整？(所有必填字段都填了？)
- [ ] 是否有交叉引用？(指向了相关条目？)
- [ ] INDEX.md 是否更新了？

## 初始化示例

```
场景: 第一次渗透测试

1. 初始化 Wiki:
   Wiki ready at /workspace/wiki.

2. 侦察 Agent 发现 192.168.1.1:3306 MySQL 5.7.38:
   → write "wiki/entities/ips/192.168.1.1.md" (使用 IP 条目模板)
   → 追加 INDEX.md: "## Entities\n- [192.168.1.1](entities/ips/192.168.1.1.md): MySQL 5.7.38:3306"

3. 情报 Agent 搜索 CVE:
   → grep_files("wiki/", "MySQL 5.7.38") → 命中 192.168.1.1
   → 搜索 CVE → 找到 CVE-2021-XXXX
   → write "wiki/entities/cves/CVE-2021-XXXX.md" (使用 CVE 条目模板)
   → 追加 INDEX.md

4. 利用 Agent 验证:
   → read_file("wiki/entities/cves/CVE-2021-XXXX.md") → 获取 PoC
   → PoC 成功 → write "wiki/discoveries/pocs/F001-poc.md"
   → write "wiki/discoveries/vuln-findings/F001.md"
   → 更新 entities/ips/192.168.1.1.md 的 Findings 列表

5. 协调员报告:
   → 收集所有 F### → write "wiki/synthesis/reports/2026-05-25-engagement.md"
   → 总结: "1 Critical finding, Wiki 共 8 条新条目"
```

---
name: secagent-knowledge
description: 安全知识管理系统 — LLM Wiki 知识库 (entities/concepts/discoveries/synthesis)、FTS5 全文搜索、跨会话记忆、知识策展
---

# SecAgent Knowledge — 知识管理与 Wiki 系统

## 能力概述

为 AI Agent 提供持久化知识管理能力。结构化 Wiki 目录 (entities/concepts/discoveries/synthesis)、FTS5 全文搜索引擎、跨会话记忆继承、自动知识策展。Agent 在安全测试过程中自动积累和检索知识。

## 调用链

```
安全发现
  → 知识采集
    → 实体提取: IP, 域名, CVE, 漏洞类型, 利用技术
    → 分类: ATT&CK 战术 / CWE 类型
  → Wiki 写入
    → entities/    — IP, 域名, CVE, 工具
    → concepts/    — 攻击方法, 防御技术, 设计模式
    → discoveries/ — 扫描结果, 漏洞发现, PoC
    → synthesis/   — 报告, 分析, 总结
  → FTS5 全文索引
    → 实时搜索: search_wiki("SQL injection UNION")
    → 历史查询: read_wiki_log (最近变更)
  → 跨会话复用
    → 记忆系统: FTS5 + cron 定时策展
    → 上下文恢复: 新会话自动加载相关条目
```

## Wiki 目录结构

```
wiki/
├── entities/
│   ├── ips/           # 目标 IP 及关联信息
│   ├── domains/       # 域名及子域名
│   ├── cves/          # CVE 条目 (CVE-2024-XXXX)
│   └── tools/         # 工具及使用记录
├── concepts/
│   ├── attack-methods/    # 攻击方法 (SQLi, XSS, BoF)
│   ├── defense-tech/      # 防御技术 (WAF, ASLR, NX)
│   └── design-patterns/   # 设计模式
├── discoveries/
│   ├── scan-results/      # 扫描结果
│   ├── vuln-findings/     # 漏洞发现
│   └── pocs/              # PoC 代码和验证结果
└── synthesis/
    ├── reports/            # 综合报告
    ├── analysis/           # 分析文档
    └── summaries/          # 摘要和结论
```

## 5 种 Wiki 操作

| 操作 | 函数 | 用途 |
|------|------|------|
| read | `read_wiki_page(path)` | 读取单个条目 |
| write | `write_wiki_page(path, content)` | 写入/更新条目 |
| search | `search_wiki(query)` | FTS5 全文搜索 |
| list | `list_wiki(dir)` | 列出目录内容 |
| log | `read_wiki_log()` | 查看最近变更 |

## 记忆系统

```
Memory System (crates/tui/src/memory/):
├── FTS5 全文搜索 — SQLite FTS5 引擎
├── 定时策展 — cron 定时清理过时条目
├── 跨会话继承 — .deepseek/memory/ 持久化
└── 上下文注入 — 新会话自动 prepend <user_memory> block
```

记忆策展触发:
- 新会话启动 → 加载最近记忆
- 超过阈值 → 清理低价值条目
- 手动触发: `memory_curator` agent

## 安全知识模板

### 实体条目 (wiki/entities/cves/CVE-YYYY-XXXX.md)
```markdown
# CVE-YYYY-XXXX
- **Product**: <product> <version>
- **Type**: <SQLi/XSS/RCE/BoF>
- **CVSS**: X.X
- **Exploit**: <PoC link or code>
- **Fixed in**: <version>
- **Discovered**: <date>
```

### 发现条目 (wiki/discoveries/vuln-findings/F###.md)
```markdown
# F###: <title>
- **Target**: [REDACTED]
- **Type**: <vuln type>
- **Severity**: Critical/High/Medium/Low
- **Status**: Confirmed/Pending/False Positive
- **PoC**: <minimal reproduction>
- **Impact**: <concrete impact>
- **Remediation**: <specific fix>
```

## 跨 Agent 知识共享

```
Agent A (recon)
  → 发现: 192.0.2.5:3306 MySQL 5.7.38
  → write_wiki_page("entities/ips/192.0.2.5.md")

Agent B (intel-gatherer)
  → search_wiki("MySQL 5.7.38")
  → 命中: 实体条目
  → 搜索 CVE + exploit-db
  → write_wiki_page("entities/cves/CVE-2021-XXXX.md")

Agent C (exploit-runner)
  → read_wiki_page("entities/cves/CVE-2021-XXXX.md")
  → 获取 PoC
  → 执行验证
  → write_wiki_page("discoveries/pocs/F001-poc.md")
```

## 关键文件

- `crates/tui/src/llmwiki_server.rs` — Wiki HTTP 服务 (1117+ 行)
- `crates/tui/src/memory/` — 记忆系统 (FTS5 + cron 策展)
- `wiki/` — Wiki 内容目录
- `crates/tui/src/tools/registry.rs` — 工具注册 (read_wiki_page, write_wiki_page, search_wiki)

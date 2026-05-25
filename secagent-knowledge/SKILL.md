---
name: secagent-knowledge
description: 安全知识管理系统 — LLM Wiki (entities/concepts/discoveries/synthesis)、FTS5 全文搜索、跨会话记忆、自动知识策展、Agent 间知识共享
version: "1.0"
source: JesusConwellpy/SecAgent-TUI (crates/tui/src/llmwiki_server.rs, crates/tui/src/memory/, wiki/)
tools: read_wiki_page, write_wiki_page, search_wiki, list_wiki, read_wiki_log, note
---

# SecAgent Knowledge — 知识管理与 Wiki 系统

## 能力概述

为 AI Agent 提供持久化、可搜索、跨会话的安全知识管理。结构化 Wiki 目录 (entities/concepts/discoveries/synthesis)、FTS5 全文搜索引擎、定时知识策展、Agent 间实时知识共享。

## 调用链 (完整端到端)

```
安全发现 (工具输出、Agent 报告、漏洞确认)
  │
  ├─→ 知识采集
  │   │
  │   ├─ 实体提取: IP, 域名, CVE, 端口, 服务版本, 利用技术
  │   ├─ ATT&CK 分类: 战术 (Recon/Execution/Persistence/...) → 技术 (TXXXX)
  │   ├─ CWE 分类: 弱点类型 (CWE-89/SQLi, CWE-79/XSS, CWE-120/BoF)
  │   └─ 严重性评估: CVSS 3.1 计算 (AV/AC/PR/UI/S/C/I/A)
  │
  ├─→ Wiki 写入 (write_wiki_page)
  │   │
  │   ├─ wiki/entities/ips/192.168.1.1.md
  │   │   └─ 关联端口、服务、漏洞
  │   │
  │   ├─ wiki/entities/cves/CVE-2024-XXXX.md
  │   │   └─ 产品、版本、CVSS、PoC、修复版本
  │   │
  │   ├─ wiki/concepts/attack-methods/sql-injection.md
  │   │   └─ 技术描述、常见 payload、绕过方法
  │   │
  │   ├─ wiki/discoveries/vuln-findings/F001-sqli-products-php.md
  │   │   └─ 目标、证据、复现步骤
  │   │
  │   └─ wiki/synthesis/reports/engagement-2026-05-25.md
  │       └─ 综合报告、严重性分布、修复建议
  │
  ├─→ FTS5 全文索引 (自动)
  │   SQLite FTS5: 每个 Wiki 文件自动分词索引
  │   支持: 精确匹配、前缀搜索、短语搜索、布尔查询
  │
  ├─→ 跨 Agent 知识共享
  │   Agent A (recon)     → write_wiki_page("entities/ips/192.0.2.5.md")
  │   Agent B (intel)     → search_wiki("MySQL 5.7.38") → 命中
  │   Agent C (exploit)   → read_wiki_page("entities/cves/CVE-2021-XXXX.md")
  │                         → 获取 PoC → 执行验证
  │
  └─→ 跨会话复用
      记忆系统加载最近条目 → 新会话注入 <user_memory> block
      → 知识策展: cron 定时清理低价值条目
```

## Wiki 操作 API

```rust
// 5 种核心操作
read_wiki_page(path: &str) -> String
    // 读取: wiki/entities/cves/CVE-2024-1234.md
    // 返回: 完整 Markdown 内容

write_wiki_page(path: &str, content: &str)
    // 写入: 创建或更新条目
    // 自动创建父目录
    // 自动触发 FTS5 索引更新

search_wiki(query: &str) -> Vec<SearchResult>
    // FTS5 全文搜索
    // 支持: "SQL injection UNION" / "CVE-2024" / "Apache 2.4"
    // 返回: [{path, title, snippet, score}]

list_wiki(dir: &str) -> Vec<DirEntry>
    // 列出目录: wiki/discoveries/vuln-findings/
    // 返回: [{name, kind(file/dir), size, modified}]

read_wiki_log() -> Vec<LogEntry>
    // 最近变更: 最后 N 次 write/update
    // 返回: [{path, action(create/update), timestamp}]
```

### FTS5 搜索语法

```
精确匹配:   "CVE-2024-1234"           → 精确 CVE 编号
前缀搜索:   "CVE-2024-*"              → 2024 年所有 CVE
短语搜索:   "SQL injection bypass"    → 含全部词的条目
布尔查询:   "Apache AND CVE NOT 2.2"  → 组合条件
字段搜索:   "type:SQLi severity:High" → (未来扩展)
```

## Wiki 目录结构与模板

```
wiki/
├── entities/           ← 实体: 具体可寻址的对象
│   ├── ips/            ← IP 地址及其关联信息
│   ├── domains/        ← 域名及子域名
│   ├── cves/           ← CVE 条目
│   └── tools/          ← 工具及使用记录
│
├── concepts/           ← 概念: 抽象知识
│   ├── attack-methods/ ← 攻击方法 (SQLi, XSS, BoF, ...)
│   ├── defense-tech/   ← 防御技术 (WAF, ASLR, NX, ...)
│   └── design-patterns/← 设计模式 (Fan-out, Mailbox, ...)
│
├── discoveries/        ← 发现: 实例化的观察
│   ├── scan-results/   ← 扫描结果 (nmap, gobuster, ...)
│   ├── vuln-findings/  ← 漏洞发现 (F001, F002, ...)
│   └── pocs/           ← PoC 代码和验证结果
│
└── synthesis/          ← 合成: 高级分析
    ├── reports/        ← 综合报告
    ├── analysis/       ← 分析文档
    └── summaries/      ← 摘要和结论
```

### CVE 条目模板

```markdown
# CVE-2024-1234
- **Product**: Apache HTTP Server 2.4.51
- **Type**: Remote Code Execution
- **CWE**: CWE-119 (Improper Restriction of Operations within the Bounds of a Memory Buffer)
- **CVSS**: 9.8 (CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H)
- **Exploit**: Available at exploit-db.com/exploits/XXXXX
- **PoC**: (attached or linked)
- **Fixed in**: Apache 2.4.52
- **Discovered**: 2024-01-15
- **Last verified**: 2026-05-25 against target [REDACTED]
```

### 漏洞发现条目模板

```markdown
# F001: SQL Injection in /products.php

- **Target**: [REDACTED]
- **Endpoint**: https://target.example.com/products.php?id=
- **Type**: SQL Injection (Union-based)
- **Severity**: High
- **CVSS**: 7.5
- **Status**: Confirmed

## Evidence
- Payload: `?id=1' UNION SELECT 1,2,3,4,5--`
- Database: MySQL 5.7.38
- Extracted: 5 columns, database 'shop_db'

## Impact
Attacker can extract all user data, including password hashes.

## Reproduction
1. Navigate to /products.php?id=1
2. Append `' UNION SELECT 1,2,3,4,5--`
3. Observe column reflection at positions 2 and 3

## Remediation
Use parameterized queries (PDO prepared statements).
```

## 记忆系统

```rust
// crates/tui/src/memory/
pub struct MemorySystem {
    db: SqliteConnection,          // SQLite FTS5 数据库
    memory_path: PathBuf,          // ~/.deepseek/memory/
    cron_interval: Duration,       // 策展间隔 (默认 24h)
    max_entries: usize,            // 最大条目数 (默认 10000)
}

impl MemorySystem {
    // 全文搜索
    pub fn search(&self, query: &str) -> Vec<MemoryEntry>

    // 写入记忆
    pub fn remember(&mut self, entry: MemoryEntry)

    // 定时策展: 清理过期/低价值条目
    pub fn curate(&mut self) -> usize  // 返回清理数量

    // 加载最近记忆 (用于新会话注入)
    pub fn load_recent(&self, limit: usize) -> Vec<MemoryEntry>
}
```

### 记忆策展规则

```
1. 超过 90 天未访问 → 标记为 stale
2. 确认度 = LOW + 超过 30 天 → 清理
3. 重复条目 (同实体 + 同发现) → 合并
4. 纯工具输出 (raw nmap scan) > 10MB → 截断到摘要
5. 关联的 case 目录已被删除 → 清理孤儿条目
```

## Agent 间知识共享示例

```
Timeline: 一次渗透测试中的 Agent 协作

T+0s:  Agent A (recon) 扫描 192.0.2.5
       → 发现: 3306/tcp MySQL 5.7.38
       → write_wiki_page("wiki/entities/ips/192.0.2.5.md")
         "192.0.2.5: MySQL 5.7.38 on 3306/tcp"

T+30s: Agent B (intel-gatherer) 启动, 搜索已有知识
       → search_wiki("MySQL 5.7.38")
       → 命中: entities/ips/192.0.2.5.md (Agent A 刚写入的)
       → search_wiki("CVE MySQL 5.7.3")
       → 命中: entities/cves/CVE-2021-XXXX.md
       → write_wiki_page("wiki/entities/cves/CVE-2021-XXXX.md")
         "CVE-2021-XXXX affects MySQL ≤5.7.39. Auth bypass. PoC available."

T+60s: Agent C (exploit-runner) 准备执行
       → read_wiki_page("wiki/entities/cves/CVE-2021-XXXX.md")
       → 获取 PoC 路径, 目标 IP, 版本确认
       → 执行 PoC → 成功 → root shell
       → write_wiki_page("wiki/discoveries/pocs/F001-cve-2021-xxxx-poc.md")
         "PoC executed successfully on 192.0.2.5:3306. Root access obtained."

T+90s: 协调员读取所有 Wiki 条目
       → 合成最终报告
       → write_wiki_page("wiki/synthesis/reports/engagement-report.md")
         "CRITICAL: MySQL 5.7.38 on 192.0.2.5 — CVE-2021-XXXX Auth Bypass → Root"
```

## 安全约束

```
1. 所有写入经过原子文件操作 (write_temp → fsync → rename)
2. 路径穿越防护: path必须规范化且不能跳出 wiki/ 根目录
3. 敏感信息脱敏: IP/域名/凭证自动替换为 [REDACTED]
4. 无外传: Wiki 内容保留在本地 workspace
5. 并发安全: 写操作通过文件锁保护
```

## 关键文件 (源仓库)

| 文件 | 行数 | 用途 |
|------|------|------|
| `crates/tui/src/llmwiki_server.rs` | 1117+ | Wiki HTTP 服务 + 工具实现 |
| `crates/tui/src/memory/` | — | FTS5 记忆系统 |
| `crates/tui/src/core/entity_graph.rs` | 200+ | 安全实体图 |
| `crates/tui/src/tools/registry.rs` | — | Wiki 工具注册 |
| `wiki/` | — | Wiki 内容目录 |

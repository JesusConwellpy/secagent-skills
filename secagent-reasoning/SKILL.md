---
name: secagent-reasoning
description: 安全侦探推理引擎 — 线索提取与交叉关联、因果图构建、12 种异常检测器、失败模式匹配与恢复、攻击链检测与假设可视化
version: "1.0"
source: JesusConwellpy/SecAgent-TUI (crates/tui/src/reasoning_engine/, crates/tui/src/causal_graph/, crates/tui/src/anomaly/, crates/tui/src/failure_patterns/)
---

# SecAgent Reasoning — 侦探推理引擎与因果图

## 能力概述

为 AI Agent 提供结构化安全推理能力。从安全发现中提取线索、构建因果图关联证据、12 种异常检测器识别攻击模式、失败模式匹配推荐恢复路径、攻击链检测与可视化。

## 调用链 (完整端到端)

```
安全发现 (工具输出、子代理报告、扫描结果)
  │
  ├─→ DetectiveEngine: 线索提取与交叉关联
  │   │
  │   ├─ 线索提取 (pattern matching + NLP)
  │   │   IP: (\d{1,3}\.){3}\d{1,3}
  │   │   Domain: [a-zA-Z0-9-]+(\.[a-zA-Z]{2,})+
  │   │   CVE: CVE-\d{4}-\d{4,}
  │   │   Port: :\d{2,5}/
  │   │   Version: \d+\.\d+(\.\d+)?
  │   │   Vuln Type: SQLi, XSS, RCE, BoF, LFI, SSRF, PrivEsc
  │   │   Technique: ROP, ret2libc, format string, heap spray
  │   │
  │   ├─ 线索关联
  │   │   时间关联: 同一时间段内的发现
  │   │   空间关联: 同一主机/服务/端口
  │   │   因果关联: A 导致 B (端口开放 → 服务运行 → 漏洞存在)
  │   │   类型关联: 同一漏洞类型的不同实例
  │   │
  │   └─ 线索优先级评分
  │       score = confidence × impact × freshness
  │
  ├─→ CausalGraph: 因果推理网络
  │   │
  │   ├─ 节点类型:
  │   │   Observation(id, fact, source, confidence)
  │   │   Hypothesis(id, claim, depends_on[], confidence)
  │   │   Verification(id, result, evidence, confidence)
  │   │   Conclusion(id, verdict, impact, confidence)
  │   │
  │   ├─ 边类型:
  │   │   caused_by(A→B):    A 是 B 的原因
  │   │   supports(A→B):     A 支撑 B 的假设
  │   │   contradicts(A→B):  A 与 B 矛盾
  │   │   depends_on(A→B):   A 依赖于 B
  │   │
  │   ├─ 置信度传播:
  │   │   新证据 → 更新所有 connected 节点
  │   │   使用贝叶斯网络更新: P(H|E) = P(E|H) × P(H) / P(E)
  │   │   矛盾检测: 两条路径得出不同结论 → 标记冲突
  │   │
  │   └─ 攻击链检测:
  │       节点序列形成完整攻击链 (Recon→Vuln→Exploit→PostExploit)
  │       → 标记为 "Attack Chain: confirmed"
  │
  ├─→ AnomalyDetector: 12 种异常检测器
  │   │
  │   ├─ PortAnomaly:        异常开放端口 (非标准/新增)
  │   ├─ ServiceAnomaly:     异常服务版本 (过时/已知漏洞)
  │   ├─ TrafficAnomaly:     异常网络流量模式
  │   ├─ FileAnomaly:        异常文件系统活动
  │   ├─ ProcessAnomaly:     异常进程行为
  │   ├─ CVEAnomaly:         CVE 版本匹配异常
  │   ├─ CredentialAnomaly:  凭证异常 (弱密码/默认凭证)
  │   ├─ ConfigAnomaly:      配置异常 (不安全配置)
  │   ├─ PatchAnomaly:       补丁缺失
  │   ├─ LogAnomaly:         审计日志异常
  │   ├─ ChainAnomaly:       攻击链完整度检测
  │   └─ BehaviorAnomaly:    行为基线偏离
  │
  ├─→ FailurePatternStore: 失败模式匹配
  │   │
  │   ├─ 已知失败模式库 (预定义 + 运行时学习)
  │   ├─ 匹配: 当前错误 vs 已知模式 → 相似度评分
  │   ├─ 推荐: 替代方案 + 成功概率估计
  │   └─ 学习: 新失败模式 → 自动入库 (带指纹去重)
  │
  └─→ 假设验证与证据链完整性检查
      ├─ 完整性: 每个 Hypothesis 有 Observation 支撑?
      ├─ 一致性: 是否存在矛盾证据?
      ├─ 可靠性: 证据来源可信?
      └─ 可重复性: 验证可复现?
```

## DetectiveEngine API

```rust
// crates/tui/src/reasoning_engine/
pub struct DetectiveEngine {
    pattern_db: Vec<CluePattern>,      // 已知线索模式
    clue_index: HashMap<String, Vec<Clue>>,  // 线索索引
}

pub struct Clue {
    pub id: String,
    pub kind: ClueKind,           // IP, Domain, CVE, Port, Service, VulnType, Technique
    pub value: String,            // "192.168.1.1", "CVE-2024-1234", "SQLi"
    pub source: String,           // "nmap scan", "agent:web-analyst", "tool:search_wiki"
    pub confidence: f32,          // 0.0 - 1.0
    pub timestamp: DateTime,
    pub related: Vec<String>,     // 关联线索 ID
}

pub enum ClueKind {
    IP, Domain, CVE, Port, Service,
    VulnType, Technique, Tool, File,
}

impl DetectiveEngine {
    // 从文本中提取线索
    pub fn extract(&self, text: &str) -> Vec<Clue>

    // 交叉关联多条线索
    pub fn cross_reference(&self, clues: &[Clue]) -> Vec<ClueRelation>

    // 构建假设
    pub fn form_hypothesis(&self, observations: &[Clue]) -> Vec<Hypothesis>

    // 验证假设
    pub fn verify(&self, hypothesis: &Hypothesis, evidence: &Evidence) -> VerificationResult
}
```

## CausalGraph API

```rust
// crates/tui/src/causal_graph/
pub struct CausalGraph {
    nodes: HashMap<String, GraphNode>,
    edges: Vec<GraphEdge>,
}

pub enum GraphNode {
    Observation { id: String, fact: String, source: String, confidence: f32 },
    Hypothesis { id: String, claim: String, depends_on: Vec<String>, confidence: f32 },
    Verification { id: String, result: bool, evidence: String, confidence: f32 },
    Conclusion { id: String, verdict: Verdict, impact: Impact, confidence: f32 },
}

pub enum GraphEdge {
    CausedBy { from: String, to: String, strength: f32 },
    Supports { from: String, to: String, weight: f32 },
    Contradicts { from: String, to: String },
    DependsOn { from: String, to: String },
}

impl CausalGraph {
    pub fn new() -> Self
    pub fn add_node(&mut self, node: GraphNode) -> String  // returns node_id
    pub fn add_edge(&mut self, edge: GraphEdge)
    pub fn propagate(&mut self, new_evidence: &Evidence)
    pub fn detect_attack_chain(&self) -> Vec<Vec<String>>  // chains of node_ids
    pub fn find_conflicts(&self) -> Vec<(String, String)>  // conflicting nodes
}
```

## 异常检测器接口

```rust
// crates/tui/src/anomaly/
pub trait AnomalyDetector: Send + Sync {
    fn name(&self) -> &str;
    fn detect(&self, data: &AnomalyInput) -> Vec<Anomaly>;
    fn baseline(&self) -> &Baseline;     // 正常基线
    fn update_baseline(&mut self, data: &AnomalyInput);
}

pub struct Anomaly {
    pub detector: String,           // 检测器名称
    pub severity: Severity,         // Low / Medium / High / Critical
    pub description: String,        // "Unusual outbound connection on port 4444"
    pub evidence: Vec<String>,      // 支撑证据
    pub recommendation: String,     // "Check for reverse shell"
    pub confidence: f32,            // 0.0 - 1.0
}

pub enum Severity {
    Low,        // 信息性 (notable but not actionable)
    Medium,     // 需要关注 (potential issue)
    High,       // 高度可疑 (likely malicious)
    Critical,   // 确定恶意 (confirmed attack)
}
```

### 检测器触发条件

```
PortAnomaly:
  触发: 端口不在已知服务映射中
  基线: 标准端口映射 (80→HTTP, 443→HTTPS, 22→SSH, ...)
  示例: "Port 4444 open — common Metasploit listener port"

ServiceAnomaly:
  触发: 服务版本匹配已知漏洞库
  基线: NVD CPE 数据库
  示例: "Apache 2.4.49 — CVE-2021-41773 Path Traversal (CVSS 9.8)"

CVEAnomaly:
  触发: product + version 精确匹配 CVE
  动作: 自动触发 intel-gatherer Agent

ChainAnomaly:
  触发: 多条线索形成 Recon→Vuln→Exploit 序列
  示例: "Port scan → SQL error → UNION extraction → password hash dump"
```

## FailurePatternStore

```rust
// crates/tui/src/failure_patterns/
pub struct FailurePatternStore {
    patterns: Vec<FailurePattern>,
    match_history: Vec<PatternMatch>,
}

pub struct FailurePattern {
    pub id: String,
    pub tool_name: String,              // 哪个工具失败
    pub error_signature: String,         // 错误模式 (regex)
    pub root_cause: String,              // 根因分析
    pub recovery_actions: Vec<String>,   // 恢复步骤 (优先级排序)
    pub success_rate: f32,               // 该恢复方案的历史成功率
    pub occurrences: u32,                // 发生次数
    pub last_seen: DateTime,
}

impl FailurePatternStore {
    pub fn shared() -> Arc<Mutex<Self>>

    // 匹配当前失败
    pub fn match_failure(&self, tool: &str, error: &str) -> Vec<&FailurePattern>

    // 记录新的失败
    pub fn record_failure(&mut self, tool: &str, error: &str, recovery: &str, success: bool)

    // 推荐恢复方案
    pub fn recommend(&self, tool: &str, error: &str) -> Option<&FailurePattern>
}
```

### 失败匹配优先级

```
1. Exact match:  同工具 + 同错误签名   → 推荐同一恢复方案
2. Similar match: 同工具 + 相似错误签名 → 推荐相似恢复方案
3. Type match:    同类型工具 + 同类型错误 → 推荐通用恢复方案
4. No match:      记录为新失败模式 → 要求人工判断
```

## 证据链完整性检查

```rust
pub struct EvidenceChainValidator;

impl EvidenceChainValidator {
    pub fn validate(chain: &EvidenceChain) -> ValidationResult {
        let mut issues = Vec::new();

        // 1. 完整性: 每个 Hypothesis 有 Observation?
        if chain.has_orphan_hypothesis() {
            issues.push("Hypothesis lacks supporting observation");
        }

        // 2. 一致性: 矛盾证据?
        if let Some(conflict) = chain.find_contradiction() {
            issues.push(format!("Contradictory evidence: {}", conflict));
        }

        // 3. 可靠性: 来源可信?
        for source in chain.sources() {
            if !source.is_trusted() {
                issues.push(format!("Untrusted source: {}", source));
            }
        }

        // 4. 可重复性: 验证可复现?
        if !chain.is_reproducible() {
            issues.push("Verification not reproducible");
        }

        ValidationResult { issues, confidence: chain.compute_confidence() }
    }
}
```

## 集成示例

```
场景: 渗透测试中发现 MySQL 5.7.38 端口 3306 开放

1. DetectiveEngine::extract(scan_output)
   → Clue{kind: Port, value: "3306"}
   → Clue{kind: Service, value: "MySQL"}
   → Clue{kind: Version, value: "5.7.38"}

2. CausalGraph::add_node(Observation { "MySQL 5.7.38 on :3306" })
   → CVEAnomaly::detect()
   → "MySQL 5.7.38 matches CVE-2021-XXXX (auth bypass)"
   → SEVERITY: HIGH

3. CausalGraph::add_node(Hypothesis { "CVE-2021-XXXX exploitable" })
   → add_edge(Supports: Observation → Hypothesis)
   → confidence: 0.7

4. Exploit 执行 → Verification { result: true, "Root shell obtained" }
   → CausalGraph::propagate(new_evidence)
   → 更新: Hypothesis confidence: 0.7 → 0.95

5. ChainAnomaly::detect()
   → "Attack chain detected: Recon(port scan) → Vuln(CVE-2021-XXXX) → Exploit(root shell)"
   → SEVERITY: CRITICAL

6. EvidenceChainValidator::validate()
   → PASS: Complete chain, no contradictions, sources trusted, result reproducible
   → confidence: 0.95
```

## 关键文件 (源仓库)

| 文件 | 行数 | 用途 |
|------|------|------|
| `crates/tui/src/reasoning_engine/` | — | 侦探推理引擎 |
| `crates/tui/src/causal_graph/` | — | 因果图系统 |
| `crates/tui/src/anomaly/` | — | 12 种异常检测器 |
| `crates/tui/src/failure_patterns/` | — | 失败模式存储与匹配 |
| `crates/tui/src/core/entity_graph/` | 200+ | 安全实体图 |
| `crates/tui/src/reasoning_tree/` | — | 推理树 (假设可视化) |
| `crates/tui/src/core/coherence.rs` | — | 自检机制 |

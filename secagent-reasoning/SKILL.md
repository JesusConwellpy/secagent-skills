---
name: secagent-reasoning
description: 安全侦探推理引擎 — 线索提取与交叉关联、因果图构建、12 种异常检测器、假设可视化与攻击链检测
---

# SecAgent Reasoning — 侦探推理引擎与因果图

## 能力概述

为 AI Agent 提供结构化安全推理能力。从安全发现中提取线索，构建因果图关联证据，运行 12 种异常检测器识别攻击模式，验证假设并可视化攻击链。

## 调用链

```
安全发现 (工具输出、Agent 报告)
  → DetectiveEngine: 线索提取
    → 关键词提取: IP, port, CVE, vuln type, technique
    → 模式匹配: 已知攻击模式库
    → 交叉关联: 多条线索 → 攻击链
  → CausalGraph: 因果推理
    → 节点: 观察 (Observation) → 假设 (Hypothesis) → 验证 (Verification) → 结论 (Conclusion)
    → 边: 因果关系 (caused_by, supports, contradicts)
    → 传播: 新证据 → 更新所有关联节点置信度
  → AnomalyDetector: 异常检测 (12 种检测器)
    → 异常网络流量模式
    → 异常文件系统活动
    → 异常进程行为
    → ...
  → FailurePatternStore: 失败模式匹配
    → 已知失败模式库
    → 匹配当前失败 → 推荐替代方案
  → 假设验证
    → 确认 (Confirmed): 证据链完整
    → 否定 (Disproven): 假设错误
    → 结论不确定 (Inconclusive): 需要更多信息
```

## 侦探引擎 (DetectiveEngine)

```
DetectiveEngine::analyze(findings: &[Finding]) -> Vec<Clue>
  ├── 线索提取: 正则 + NLP 模式匹配
  ├── 线索分类: IP/domain/CVE/technique/vuln_type
  ├── 交叉关联: 多条线索之间的关联关系
  │   ├── 时间关联: 同一时间段
  │   ├── 空间关联: 同一主机/服务
  │   └── 因果关联: A 导致 B
  └── 线索优先级: 置信度 × 影响度
```

## 因果图 (CausalGraph)

```
节点类型:
  Observation   — 原始发现 (atomic fact)
  Hypothesis    — 可测试的假设 (testable, falsifiable)
  Verification  — 验证结果 (pass/fail/inconclusive)
  Conclusion    — 最终结论 (confirmed/disproven)

边类型:
  caused_by     — A 是 B 的原因
  supports      — A 支持 B 的假设
  contradicts   — A 与 B 矛盾
  depends_on    — A 依赖于 B

传播规则:
  新证据 added → 所有 connected 节点重新评估
  置信度变化 → 传播到所有 downstream 节点
  矛盾检测 → 标记冲突节点, 需要更多证据
```

## 12 种异常检测器

| 检测器 | 检测内容 | 触发条件 |
|--------|---------|---------|
| PortAnomaly | 异常开放端口 | 非标准端口 / 新增端口 |
| ServiceAnomaly | 异常服务版本 | 过时版本 / 已知漏洞版本 |
| TrafficAnomaly | 异常网络流量 | 突发流量 / 异常协议 |
| FileAnomaly | 异常文件活动 | 权限变更 / 敏感文件访问 |
| ProcessAnomaly | 异常进程行为 | 权限提升 / 隐藏进程 |
| CVEAnomaly | CVE 相关性 | 版本匹配 / PoC 可用 |
| CredentialAnomaly | 凭证异常 | 弱密码 / 默认凭证 / 泄露 |
| ConfigAnomaly | 配置异常 | 不安全配置 / 默认配置 |
| PatchAnomaly | 补丁缺失 | 已知漏洞未修补 |
| LogAnomaly | 日志异常 | 异常登录 / 审计日志异常 |
| ChainAnomaly | 攻击链检测 | 多条线索形成完整攻击链 |
| BehaviorAnomaly | 行为异常 | 偏离基线行为模式 |

## 失败模式存储 (FailurePatternStore)

```
FailurePatternStore:
├── 已知失败模式库 (预定义 + 运行时学习)
├── 匹配算法: 当前错误 vs 已知模式
├── 推荐: 替代方案 + 成功概率
└── 学习: 新失败模式 → 自动入库

匹配优先级:
  1. 精确匹配: 同工具 + 同错误
  2. 相似匹配: 同工具 + 相似错误
  3. 类型匹配: 同类型工具 + 同类型错误
```

## 证据链完整性检查

```
EvidenceChainValidator::validate(chain) -> ValidationResult
  ├── 完整性检查: 每个 Hypothesis 有 Observation 支撑?
  ├── 一致性检查: 有矛盾证据吗?
  ├── 可靠性检查: 证据来源可信吗?
  └── 可重复性检查: 验证可复现吗?
```

## 关键文件

- `crates/tui/src/reasoning_engine/` — 侦探推理引擎
- `crates/tui/src/causal_graph/` — 因果图系统
- `crates/tui/src/anomaly/` — 异常检测器 (12 种)
- `crates/tui/src/failure_patterns/` — 失败模式存储与匹配
- `crates/tui/src/core/entity_graph/` — 安全实体图
- `crates/tui/src/reasoning_tree/` — 推理树 (假设可视化)

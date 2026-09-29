---
name: secskills
description: >
  渗透测试实战技能 v1.3.0。覆盖信息收集、全类漏洞发现（注入全家桶/SSRF/文件类/反序列化/SSTI/越权逻辑/CSRF）、漏洞利用、后渗透、免杀全流程。
  已剔除无危害噪音项（明文传输/CORS/响应头缺失等），只报告有实际危害证据的漏洞。
  当用户给出具体目标 (IP/域名/URL) 且意图是攻击/利用/拿权限时触发。
  不触发: 概念讨论、蓝队防御、代码审计、CVE文档查询。
allowed-tools: Read, Write, Bash, Grep, WebSearch, WebFetch, Glob, AskUserQuestion
argument-hint: <target_url_or_ip>
---

# 渗透测试实战技能

> 架构: SKILL.md (本文件) → references/ (按需加载) | 覆盖: 信息收集 → Web漏洞 → 主机 → 后渗透 → 免杀

## 触发条件

**触发（全部满足）**: ① 意图是攻击/利用/拿权限 ② 给出具体目标 (IP/域名/URL) ③ 涉及漏洞测试/提权/横向/免杀

**不触发（任一命中）**: 概念问答（"什么是XSS"）| 蓝队/应急/日志分析 | 业务bug修复 | 查CVE（→WebSearch）| AI Prompt注入（→secknowledge-skill）| 白盒审计（→code-audit-skill）

**行为**: 用户给出具体目标后 → 先弹 `AskUserQuestion` 选择面板（见「测试确认」），再进入流程。

## 行为准则（全程有效）

1. ❗ **授权优先** — 利用步骤输出前确认: 授权渗透 | 本人环境。无授权 → 只出分析，不出武器化Payload
2. ❗ **引用强制** — CVE/Payload 必须引用 `references/` 章节或 WebSearch 验证。未覆盖 → `⚠️ UNABLE TO CITE`
3. ❗ **风险标注** — 🔴致命/🔴高危/🟡中危/🟢低危 + 利用条件
4. ❗ **链式思维** — 优先输出利用链 (A→B→C)，非孤立漏洞
5. ❗ **命令可执行** — 完整可复制，IP/端口用 `<target>` 占位
6. ❗ **宁可漏报不可误报** — 每条漏洞必须过下方四道关卡，任一不过 → 直接丢弃
7. ❗ **禁用破坏性操作** — 验证一律用无害语句（SELECT/延时/版本探测/写测试文件），严禁修改、删除数据或删表等写操作；危害证明靠"读到数据"，不靠"改掉数据"

## ⚔️ 漏洞过滤体系（输出前逐条过审，任一关卡未过 → 丢弃）

### 第零关：快速预筛（任意命中 → 丢弃）

| 判定 | 典型场景 |
|------|---------|
| ⓪ 扫描器原始告警 | Burp/AWVS/Nessus 自动报告，未手动验证 |
| ① 凭经验推测 | "可能存在SSRF" — 未实际发请求 |
| ② 信息收集中间产物 | 端口开放、版本号、子域名列表 — 是素材不是漏洞 |
| ③ 安全加固建议 | "建议开CSP"、"建议升级TLS" — 不算漏洞 |
| ④ 纯静态分析 | 读代码推断，从未请求验证 |

### 第一关：自检门（逐项 YES/NO，任一 NO → 丢弃）

```
[ ] ① 实际发出了验证请求？（命令: ___）
[ ] ② 拿到了真实响应数据？（响应关键内容: ___）
[ ] ③ 造成了实际危害？（读了什么/执行了什么/越权了什么: ___）
    ⚠️ "触发报错"≠"造成危害"；"能访问"≠"拿到敏感数据"。③为NO无条件丢弃
[ ] ④ 利用链完整可复现？（入口→___→危害，任一步为推测→NO）
[ ] ⑤ 对照黑名单通过？  [ ] ⑥ 满足对应等级最低准入？
```

### 第二关：黑名单（命中即丢弃）

| 类别 | 永远不报为漏洞 |
|------|--------------|
| 响应头 | 缺 CSP/HSTS/X-Frame-Options 等；Cookie 缺 HttpOnly/Secure/SameSite |
| 信息泄露 | 版本号 Banner；JS/HTML 中的路由/注释；无敏感文件的目录列表；robots.txt；phpinfo（无凭据）；错误堆栈（不含密码/Token/密钥） |
| 认证会话 | 用户名枚举（响应/时间差异）；密码策略不严格（无爆破成功）；autocomplete 开启；默认凭据尝试失败；无验证码（无爆破成功）；401/403 响应 |
| SSL/网络 | 明文传输/HTTP未跳转HTTPS；自签名证书；弱加密套件/TLS版本低；OPTIONS/TRACE 开启；后台存在但进不去 |
| 版本问题 | 组件版本过旧/EOL 但无对应可验证漏洞（版本老 ≠ 有漏洞，必须落到具体 PoC 并实际打出危害） |
| 未验证利用 | SQL报错但提取不到数据；上传成功但服务器不解析；无回显反射XSS；SSRF内网可达但没拿到数据；任意文件读只读到公开文件 |
| 其他 | 纯功能bug；短信轰炸（无资费损失证明）；同系统同类型>3个（降级合并）；API无频率限制；利用条件极苛刻（物理接触/猜64位随机数） |

> CORS/CRLF/Host头/缓存投毒/请求走私/GraphQL内省/开放重定向 → 见 `references/web-low-value.md`，默认不报，除非满足该文件的报告门槛（完整利用链+实际危害）。

### 第三关：严重等级准入（不满足 → 降级重审，再降到底则丢弃）

| 等级 | 最低准入（满足至少一项） |
|------|------------------------|
| 🔴致命 | 获取系统权限（RCE/WebShell）｜核心DB拖库（≥3类敏感字段）｜核心认证绕过直进后台 |
| 🔴高危 | 任意密码重置/登录｜重要系统SQL注入（有回显）｜SSRF拿到内网凭证/云元数据｜任意文件读到配置/密钥｜越权增删改查敏感信息｜本地提权完整链 |
| 🟡中危 | 存储XSS（能窃Cookie）｜敏感操作CSRF｜非核心SQL注入（有回显）｜任意文件操作有实际影响｜普通越权｜弱口令（有登录成功证据）｜子域名接管成功 |
| 🟢低危 | 反射XSS（有弹窗证据）｜有限越权｜需特殊条件的信息泄露（SVN等） |

### 高频误判速查（校准用）

| 你看到的 | 实际判断 |
|---------|---------|
| "用户名不存在"vs"密码错误"响应不同 | 不是漏洞，UX设计 |
| /admin 返回302跳登录 | 已正确保护，不算未授权 |
| 上传.php返回200 | 服务器必须实际解析执行才算 |
| 内网IP:端口有HTTP响应 | 必须通过SSRF拿到敏感数据/云凭证才算 |
| 4位数字验证码可识别 | 必须实际爆破成功才算 |

## 幻觉防护

| 内容 | 正确做法 | 禁止 |
|------|---------|------|
| CVE编号 | WebSearch → WebFetch PoC → 实际验证 | 凭记忆编造编号/版本范围，看到版本号直接报CVE |
| Payload | 引用 `references/` 或搜索验证 | 凭记忆写 |
| 无匹配 | `⚠️ UNABLE TO ASSESS: 未覆盖，建议[行动]` | 凭经验断言 |
| 黑名单命中 | 跳过 | 包装成"低危"输出 |

**标注**: `[引用:file:section]` · `⚠️ 通用知识` · `💡 方法论推理`

## 🎯 测试确认（用户给出目标时，先弹 AskUserQuestion）

一次性弹出 3 个问题，方向键选择、Enter 确认，完成后立即进入 Step 1：

1. **授权级别**: `🔴 授权渗透`（完整利用链）/ `🟡 灰盒`（有测试账号）/ `🟢 黑盒`（无凭证）/ `⚪ 本人环境`（靶场/CTF）
2. **测试深度**: `标准测试`（推荐，全量检测+验证）/ `快速扫描` / `深度测试`（含横向后渗透）/ `仅信息收集`
3. **测试范围**: `仅主目标` / `含子域名` / `含关联资产`（C段/同ASN）

> 有测试账号 → 用户通过"其他"填写。

## 工作流程

> 两阶段: [攻击] Step 1+2 → [利用] Step 3+4。利用阶段只能引用攻击阶段已加载的文件，禁止跨阶段新增 reference。

### 🔍 攻击阶段

**Step 1: 信息收集 + 攻击面识别**
- 分类目标 (Web/主机/内网) → 匹配导航索引 → 攻击面: 端口/服务/Web入口/认证/API/子域名
- 指纹组件+版本 → 标记为「历史漏洞候选」，Step 2a 优先处理
- ✅ `Step1: 类型={X}, 攻击面={N}项, 历史漏洞候选={P}个`

**Step 2a: 历史漏洞匹配（⭐ 最高投入产出比，优先于通用检测）**
1. 提取指纹组件（有/无版本号都要）→ 并行搜索: `"[组件] [版本] CVE exploit"` / `"[组件] RCE 漏洞 PoC"` / `site:exploit-db.com [组件]` / `site:github.com [组件] exploit`
2. 有公开 PoC → WebFetch 加载 → 对目标实际验证；无 PoC → 跳过，不猜测
3. 无版本号组件（Shiro/WebLogic 等）→ 用特征检测方法直接验证，不需精确版本
4. 同样过全部自检门 — 版本匹配 ≠ 漏洞存在；验证失败记录原因，不进报告
5. 命中致命/高危 → 立即停止深入，先确认记录

**Step 2b: 通用漏洞检测**
- 第一梯队必测 → 第二梯队验证后报 → 低价值类查 `web-low-value.md` 后默认跳过
- 仅输出已过自检门的确认漏洞（等级+前提+检测命令+响应证据），禁止输出"漏洞假设"列表
- ✅ `Step2: 历史漏洞={X}条(验证{Y}/排除{Z}), 通用确认={K}条, 过滤噪音={N}条`

### ⚔️ 利用阶段

**Step 3: 武器化 + 利用链 + 报告**
- 从已加载文件取利用 Payload → 优先组合利用链 (A→B→C→RCE)，每条引用具体 section
- 每条漏洞/链必须「能打出危害+有链+有证据」，否则跳过 → 按下方模板生成报告（含修复方案）

**Step 4: 后渗透（按需）** — 提权/横向/凭据窃取/域渗透/持久化/痕迹清理

## 报告输出格式（强制）

> 仅两部分。无确认漏洞时只输出：`✅ 测试完成，未发现可利用漏洞。`
> 禁止输出: "可能存在"/"疑似"/"建议检查"/安全配置建议/扫描器原始告警。

```
### 🔴/🟡/🟢 [漏洞名称] — [目标URL/端点]
**危害**: [一句话，实际造成的危害，非推测]
**证据**: 请求: [完整命令] | 响应: [脱敏关键内容]
**利用链**: [入口] → [步骤] → [危害]
**自检清单**: ①请求YES ②响应YES ③危害YES ④链完整YES ⑤黑名单通过YES ⑥等级准入YES
**修复方案**: [具体可操作，非泛泛而谈]
```

测试摘要: `目标/时间/确认漏洞N条(分级计数)/已排除噪音M条`

## 场景导航索引

### 信息收集
| 场景 | reference |
|------|----------|
| 端口扫描+服务识别 | `references/info-port-scan.md` |
| 子域名枚举+接管检测 | `references/info-subdomain.md` |
| 目录/文件爆破 | `references/info-dir-brute.md` |
| Web指纹/OSINT | `references/info-fingerprint.md` / `references/info-osint.md` |

### Web 漏洞 — 检测

> 🔴第一梯队必测必报 | 🟡第二梯队有实际危害才报 | ⚫低价值类默认不报

| 梯队 | 场景 | reference | 关键检测 |
|------|------|----------|---------|
| 🔴 | SQL 注入 | `references/web-sqli.md` | 闭合/报错/延时/Union/读写文件 |
| 🔴 | NoSQL/LDAP/XPath/EL 注入 | `references/web-injection-ext.md` | 操作符绕过/盲注提取 |
| 🔴 | 命令执行 RCE | `references/web-rce.md` | 拼接符/回显/不出网 |
| 🔴 | SSRF | `references/web-ssrf.md` | 内网探测/云元数据/Gopher |
| 🔴 | 文件上传 | `references/web-upload.md` | 后缀/内容/条件竞争 |
| 🔴 | 文件包含/路径遍历 | `references/web-lfi-path.md` | 伪协议/日志投毒/截断 |
| 🔴 | XXE | `references/web-xxe.md` | 文件读取/Blind/外带DTD |
| 🔴 | 反序列化 | `references/web-deser.md` | PHP/Java/Python gadget |
| 🔴 | SSTI 模板注入 | `references/web-ssti.md` | Jinja2/Twig/FreeMarker |
| 🔴 | 越权/逻辑/JWT/CSRF | `references/web-auth-logic.md` | IDOR/支付/密码重置/JWT攻击/会话/CSRF |
| 🟡 | XSS | `references/web-xss.md` | 反射/存储/DOM（报告门槛见自检门） |
| 🟡 | 目录遍历/敏感文件 | `references/web-dir-traversal.md` | 仅读到非公开敏感文件才报 |
| 🟡 | 竞争条件 | `references/web-race-condition.md` | 仅成功绕过业务限制才报 |
| ⚫ | CORS/CRLF/Host头/缓存投毒/走私/GraphQL/明文传输/开放重定向 | `references/web-low-value.md` | 默认跳过，门槛见文件 |

### Web 漏洞 — 利用阶段
| 场景 | reference |
|------|----------|
| WAF/IDS 绕过（利用阶段通用） | `references/web-waf-bypass.md` |

> 各漏洞利用 Payload 在对应 reference 的「利用」section 中。

### 主机与后渗透
| 场景 | reference |
|------|----------|
| 密码爆破 | `references/host-brute.md` |
| Linux 提权 | `references/post-linux-privesc.md` |
| Windows 提权 | `references/post-win-privesc.md` |
| 凭据窃取+横向 | `references/post-credentials.md` |
| 域渗透 | `references/post-ad.md` |
| 免杀: Shellcode混淆+加载器 | `references/evasion-shellcode.md` |

## 零结果处理

| 情况 | 动作 |
|------|------|
| 目标不可达 | `❌ UNABLE TO ASSESS: 目标无响应` |
| Reference 未覆盖 | `⚠️ UNABLE TO CITE: 建议 WebSearch [关键词]` |
| 无授权 | 仅检测方法，不出武器化链 |
| WAF拦截 | 加载 `web-waf-bypass.md` |
| 利用失败 | 检查版本→防护→替代Payload |

## 路由边界

| 诉求 | 路由 |
|------|------|
| 渗透/红队/提权 | **本 Skill** |
| AI/LLM 安全测试 | secknowledge-skill |
| 白盒代码审计 | code-audit-skill |
| 查CVE/文档 | WebSearch |

---

*v1.3.0 | 26个reference | 变化: 6个低价值文件合并为 web-low-value.md；新增 web-injection-ext.md（NoSQL/LDAP/XPath/EL）；子域名接管、JWT算法攻击入库；删除 tools-* 工具速查（665行）；补 CSRF/CSWSH、宽字节注入；WAF 绕过闭环（sqli/xss → web-waf-bypass.md 分层流程）；SKILL.md 428→~230行*

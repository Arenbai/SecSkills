# 🔐 SecSkills — 渗透测试实战技能

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-orange)](https://claude.ai/code)
[![Version](https://img.shields.io/badge/version-1.3.0-green.svg)]()
[![Stars](https://img.shields.io/badge/⭐_Star-支持本项目-yellow)]()
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)]()

> 专为 **Claude Code / WorkBuddy** 等支持 `SKILL.md` 的 Agent 环境打造的专业渗透测试技能包。覆盖信息收集 → 全类漏洞发现 → 漏洞利用 → 后渗透 → 免杀全流程。对 Agent 说出目标，自动弹出方向键交互面板，即刻开始实战测试。
>
> 💡 **v1.3.0 已剔除无危害噪音项**（明文传输 / CORS / 响应头缺失等配置类问题），只报告有实际危害证据的漏洞；所有验证统一使用无害语句，不执行任何修改或删除数据的操作。

## ⭐ 支持项目

如果这个项目对你有帮助，请点个 **Star** ⭐ 支持一下！也欢迎 **Fork**、**提 Issue**、**PR 贡献**。

---

## 🚀 快速开始

### 安装

```bash
# 克隆到当前目录
git clone https://github.com/Arenbai/SecSkills.git .

# 克隆到 Claude Code 用户级 skills 目录（全局生效）
git clone https://github.com/Arenbai/SecSkills.git ~/.claude/skills/secskills

# 或者放到项目本地 .claude/skills/ 下（仅当前项目生效）
git clone https://github.com/Arenbai/SecSkills.git .claude/skills/secskills

# WorkBuddy 用户级 skills 目录
git clone https://github.com/Arenbai/SecSkills.git ~/.workbuddy/skills/secskills
```

### 使用

在对话中直接说：

```
对 https://target.com 做渗透测试
```

Agent 会弹出 **↑↓ 方向键选择面板**，确认授权级别、测试深度、范围后自动开始。

---

## 📋 功能架构

```
SecSkills/
├── SKILL.md              # 技能主文件（触发条件 + 行为准则 + 漏洞过滤四道关卡 + 两阶段工作流 + 导航索引）
├── README.md            # 本说明
├── LICENSE
└── references/           # 知识库（26 个专项文件）
    ├── info-*.md         # 信息收集（端口/子域名/目录/指纹/OSINT）
    ├── web-*.md          # Web 漏洞检测（SQL注入→扩展注入→XSS→RCE→SSRF→反序列化...共 15 类）
    ├── post-*.md         # 后渗透（Linux提权/Windows提权/凭据/域渗透）
    ├── host-*.md         # 主机服务（密码爆破）
    └── evasion-*.md      # 免杀规避（Shellcode混淆+加载器）
```

---

## 🎯 覆盖范围

### 🔍 信息收集

- 端口扫描 + 服务识别 (Nmap)
- 子域名枚举（字典+证书透明度+搜索引擎）+ **子域名接管检测**
- 目录/文件爆破（Gobuster/ffuf）
- Web 指纹识别 / OSINT

### 🎯 Web 漏洞

> 🔴 第一梯队（必测必报）｜🟡 第二梯队（有实际危害才报）｜⚫ 低价值类（默认跳过，满足利用链门槛才报）

| 层级 | 漏洞类型      | 包含内容 / 对应文件                                            |
| ---- | ------------ | ------------------------------------------------------------- |
| 🔴   | SQL 注入     | Union/报错/盲注/DNS外带/堆叠/Header注入 → `web-sqli.md`        |
| 🔴   | NoSQL/LDAP/XPath/EL 注入 | 操作符绕过/盲注提取/算法表达式注入 → `web-injection-ext.md`（v1.3.0 新增） |
| 🔴   | 命令执行 RCE | 命令拼接/绕过/参数注入/不出网                                  |
| 🔴   | SSRF         | 内网探测/云元数据/Gopher/DNS重绑定                             |
| 🔴   | 文件上传     | 后缀绕过/内容绕过/条件竞争/云存储利用                          |
| 🔴   | 文件包含/路径遍历 | LFI/伪协议/日志投毒/Session包含/截断                        |
| 🔴   | XXE          | 文件读取/Blind OOB/XInclude/SSRF组合                          |
| 🔴   | 反序列化     | PHP/Java/Python/.NET/Node.js                                  |
| 🔴   | SSTI         | Jinja2/FreeMarker/Velocity/Smarty/Twig/ERB                    |
| 🔴   | 越权/逻辑/JWT | IDOR/支付篡改/竞争条件/验证码缺陷/JWT算法攻击/API鉴权绕过 → `web-auth-logic.md` |
| 🟡   | XSS          | 反射/存储/DOM/CSP绕过/Mutation XSS                             |
| 🟡   | 目录遍历/敏感文件 | 路径穿越/目录列表/编码绕过/配置源码密钥读取                 |
| 🟡   | 竞争条件     | 并发绕过/TOCTOU/秒杀/优惠券/库存竞争                           |
| ⚫   | CORS/CRLF/Host头/缓存投毒/请求走私/GraphQL/明文传输/开放重定向 | 合并为 `web-low-value.md`（v1.3.0 合并 6 个专项），含"何时才值得报"的利用链门槛 |
| —    | WAF 绕过     | 编码/分块/HPP/协议走私（利用阶段通用）→ `web-waf-bypass.md`    |

### ⚔️ 后渗透

- **Linux 提权**: SUID/Capabilities/Cron/内核Exploit
- **Windows 提权**: Token窃取/服务提权/UAC绕过/PrintSpoofer
- **凭据窃取**: Mimikatz/内存转储/DPAPI/浏览器密码
- **横向移动**: PTH/PTT/WMI/WinRM/PSExec
- **域渗透**: Kerberoasting/AS-REP/DCSync/Golden Ticket/PetitPotam

### 🛡️ 免杀规避

- Shellcode 混淆 + 多语言加载器
- MSFvenom 生成 + 编码器链

---

## 🖥️ 交互流程

```
用户: /secskills
Agent: （正常待命）

用户: 对 https://target.com 做渗透测试
       ↓
Agent: ┌─────────────────────────────────┐
       │   ↑↓ 方向键选择                   │
       │   ① 授权: 授权渗透/灰盒/黑盒/本人  │
       │   ② 深度: 标准/快速/深度/信息收集  │
       │   ③ 范围: 主目标/含子域名/关联资产 │
       └─────────────────────────────────┘
       ↓
Agent: ✅ 开始标准渗透测试 →
       Step 1: 信息收集（端口/子域名/指纹）
       Step 2: 漏洞发现（指纹匹配历史漏洞 → 通用检测，按攻击面加载）
       Step 3: 漏洞利用（构建利用链 A→B→C → 报告）
       Step 4: 后渗透（按需触发）
```

---

## 🔒 安全红线

- 🚫 无授权不输出武器化 Payload
- 🚫 **禁止使用破坏性操作**：验证一律用无害语句（SELECT/延时/版本探测/写测试文件），严禁修改、删除数据或删表
- 🚫 PoC 仅做验证，读取少量数据证明可行性，不污染/破坏目标系统
- ✅ 报告中所有敏感数据必须脱敏
- ✅ 高危漏洞立即停止深入利用，先与客户确认
- ✅ 无危害噪音项（明文传输/CORS/响应头缺失/开放重定向等）一律不报告，仅在有完整利用链并造成实际危害时才计入

---

## 📌 适用场景

| 场景             | 支持 |
| ---------------- | ---- |
| Web 应用渗透测试 | ✅    |
| API 安全测试     | ✅    |
| 内网渗透         | ✅    |
| 红队演练         | ✅    |
| CTF 竞赛         | ✅    |
| 源码审计         | ⚠️ 白盒审计走 code-audit-skill |
| 移动端安全测试   | ❌ 不在本 Skill 范围 |

---

## 🔗 依赖

- [Claude Code](https://claude.ai/code)（CLI 或 IDE 插件）/ [WorkBuddy](https://www.workbuddy.cn)（已内置 SKILL.md 加载）
- 无额外工具依赖（Agent 会自动调用 Bash / WebSearch 等内置工具；实战类工具如 nmap/sqlmap 按需本地安装）

---

## 🤝 贡献指南

欢迎贡献！你可以：

- 🐛 **提 Issue** — 报告 Bug、建议新功能、反馈使用体验
- 🔀 **Fork + PR** — 新增漏洞模块、优化现有 Reference、完善文档
- ⭐ **Star** — 觉得有用就点亮星星，让更多人看到

---

## 📄 声明

本技能仅供合法授权的安全测试使用。使用者应确保获得目标系统所有者的书面授权。作者不对任何非法使用行为承担责任。

---

## 📜 License

MIT © 2025 Arenbai

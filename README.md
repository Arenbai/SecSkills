# 🔐 SecSkills — 渗透测试实战技能

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-orange)](https://claude.ai/code)
[![Version](https://img.shields.io/badge/version-1.3-green.svg)]()
[![Stars](https://img.shields.io/badge/⭐_Star-支持本项目-yellow)]()
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)]()

> 专为 **Claude Code** 打造的专业渗透测试技能包。覆盖信息收集 → 漏洞发现 → 漏洞利用 → 后渗透 → 免杀全流程。对 Claude 说出目标，自动弹出方向键交互面板，即刻开始实战测试。

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
```

### 使用

在 Claude Code 对话中直接说：

```
对 https://target.com 做渗透测试
```

Claude 会弹出 **↑↓ 方向键选择面板**，确认授权级别、测试深度、范围后自动开始。

---

## 📋 功能架构

```
SecSkills/
├── CLAUDE.md             # 独立提示词（可直接用作项目 .claude/CLAUDE.md）
├── SKILL.md              # 技能主文件（工作流 + 导航索引）
└── references/           # 知识库（36 个专项文件）
    ├── info-*.md         # 信息收集（端口/子域名/目录/指纹/OSINT）
    ├── web-*.md          # Web 漏洞（SQL注入→XSS→RCE→SSRF→反序列化...共 19 类）
    ├── post-*.md         # 后渗透（Linux提权/Windows提权/凭据/域渗透）
    ├── host-*.md         # 主机服务（密码爆破）
    ├── evasion-*.md      # 免杀规避（Shellcode混淆+加载器）
    └── tools-*.md        # 工具速查（nmap/sqlmap/msf/hydra/impacket/ffuf）
```

---

## 🎯 覆盖范围

### 🔍 信息收集
- 端口扫描 + 服务识别 (Nmap)
- 子域名枚举（字典+证书透明度+搜索引擎）
- 目录/文件爆破（Gobuster/ffuf）
- Web 指纹识别 / OSINT

### 🎯 Web 漏洞（19 类）

| 漏洞类型 | 包含内容 |
|---------|---------|
| SQL 注入 | Union/报错/盲注/DNS外带/堆叠/二次注入/Header注入/NoSQL |
| XSS | 反射/存储/DOM/CSP绕过/Mutation XSS |
| 命令执行 | 命令拼接/绕过/参数注入/代码执行 |
| SSRF | 内网探测/云元数据/Gopher协议/DNS重绑定/编码绕过 |
| 文件上传 | 后缀绕过/内容绕过/条件竞争/云存储利用 |
| 文件包含 | LFI/RFI/伪协议/日志投毒/Session包含 |
| **🆕 目录遍历** | **路径穿越/目录列表泄露/编码绕过/敏感文件读取** |
| XXE | 文件读取/Blind OOB/XInclude/SSRF组合 |
| 反序列化 | PHP/Java/Python/.NET/Node.js |
| 越权/逻辑 | IDOR/支付篡改/竞争条件/验证码缺陷 |
| **🆕 竞争条件** | **并发绕过/TOCTOU/秒杀/优惠券/库存竞争** |
| SSTI | Jinja2/FreeMarker/Velocity/Smarty/Twig/ERB |
| HTTP 请求走私 | CL.TE/TE.CL/H2降级 |
| Host 头注入 | 密码重置投毒/缓存投毒/内网扫描 |
| 缓存投毒 | 非键化输入/响应篡改/CDN层投毒 |
| CORS | Origin反射/Credentials/子域名XSS链 |
| CRLF | 响应头注入/会话固定/响应拆分 |
| GraphQL | 内省/批量查询/深度递归 |
| WAF 绕过 | 编码/分块/HPP/协议走私 |

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
Claude: （正常待命）

用户: 对 https://target.com 做渗透测试
       ↓
Claude: ┌─────────────────────────────────┐
        │   ↑↓ 方向键选择                   │
        │   ① 授权: 授权渗透/灰盒/黑盒/本人  │
        │   ② 深度: 标准/快速/深度/信息收集  │
        │   ③ 范围: 主目标/含子域名/关联资产 │
        └─────────────────────────────────┘
       ↓
Claude: ✅ 开始标准渗透测试 →
        Step 1: 信息收集（端口/子域名/指纹）
        Step 2: 漏洞发现（按攻击面加载检测知识）
        Step 3: 漏洞利用（构建利用链 A→B→C）
        Step 4: 后渗透（按需触发）
```

---

## 📝 更新日志

### v1.3 (2026-06-01)

**🆕 新增 2 个专项模块：**

| 新增模块 | 文件 | 核心内容 |
|---------|------|---------|
| 竞争条件/并发 | `web-race-condition.md` | TOCTOU原理、Turbo Intruder并发脚本、点赞/收藏刷票、优惠券重复领取、余额竞争、秒杀超卖、文件上传竞态、多步流程绕过、验证码并发爆破、asyncio高并发模板 |
| 目录遍历 | `web-dir-traversal.md` | 目录列表泄露(Apache/Nginx/IIS)、路径穿越检测、编码绕过(URL/Unicode/UTF-8)、后缀限制绕过、CVE-2021-41773/42013、IIS分号截断、自动化检测脚本 |

**🔧 增强 4 个已有模块：**

| 增强模块 | 新增内容 |
|---------|---------|
| CORS | 子域名XSS链式利用、CORS+CSRF组合、WebSocket CORS绕过、预检请求绕过、自动化检测脚本 |
| 缓存投毒 | Varnish/Cloudflare/Fastly CDN投毒、Fat GET投毒、缓存键归一化攻击、未键化Cookie投毒 |
| Host 头注入 | Django/Flask/Laravel框架投毒、多域名探测、Host头+SSRF内网扫描增强 |
| CRLF 注入 | CRLF→XSS→Cookie窃取完整链、Header反射链、CRLF+SSRF组合 |

**📊 数据：** Web漏洞 17→19 类 | 专项文件 34→36 个

### v1.2 (2025-05-31)
- 首次发布，覆盖 17 类 Web 漏洞 + 后渗透 + 免杀
- 34 个专项 reference 文件
- AskUserQuestion 方向键交互选择面板

---

## 🔒 安全红线

- 🚫 无授权不输出武器化 Payload
- 🚫 禁止在生产环境执行破坏性操作
- 🚫 PoC 仅做验证，读取少量数据证明可行性
- ✅ 报告中所有敏感数据必须脱敏
- ✅ 高危漏洞立即停止深入利用，先与客户确认

---

## 📌 适用场景

| 场景 | 支持 |
|------|------|
| Web 应用渗透测试 | ✅ |
| API 安全测试 | ✅ |
| 内网渗透 | ✅ |
| 移动端安全测试 | ✅ |
| 源码审计 | ✅ |
| CTF 竞赛 | ✅ |
| 红队演练 | ✅ |

---

## 🔗 依赖

- [Claude Code](https://claude.ai/code)（CLI 或 IDE 插件）
- 无额外工具依赖（Claude 会自动调用 Bash/WebSearch 等内置工具）

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

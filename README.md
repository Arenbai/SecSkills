# SecSkills — 渗透测试实战技能

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-orange)](https://claude.ai/code)
[![Version](https://img.shields.io/badge/version-1.2-green.svg)]()

渗透测试实战技能包，专为 **Claude Code** 设计。覆盖信息收集 → 漏洞发现 → 漏洞利用 → 后渗透 → 免杀全流程。当你对 Claude 说"帮我测试这个站"时，它会弹出方向键选择面板确认授权和范围，然后自动执行完整的渗透测试工作流。

## 快速开始

### 安装

```bash
# 克隆到当前目录
git clone https://github.com/Arenbai/SecSkills.git .

# 克隆到 Claude Code skills 目录
git clone https://github.com/Arenbai/SecSkills.git ~/.claude/skills/secskills

# 或者放到项目本地 .claude/skills/ 
git clone https://github.com/Arenbai/SecSkills.git .claude/skills/secskills
```

### 使用

在 Claude Code 对话中直接说：

```
对 https://target.com 做渗透测试
```

Claude 会弹出 **↑↓ 方向键选择面板**，确认授权级别、测试深度、范围后自动开始。

## 功能架构

```
SecSkills/
├── SKILL.md              # 技能主文件（工作流 + 导航索引）
└── references/           # 知识库（34个专项文件）
    ├── info-*.md         # 信息收集（端口/子域名/目录/指纹/OSINT）
    ├── web-*.md          # Web 漏洞（SQL注入→XSS→RCE→SSRF→反序列化...共17类）
    ├── post-*.md         # 后渗透（Linux提权/Windows提权/凭据/域渗透）
    ├── host-*.md         # 主机服务（密码爆破）
    ├── evasion-*.md      # 免杀规避（Shellcode混淆+加载器）
    └── tools-*.md        # 工具速查（nmap/sqlmap/msf/hydra/impacket/ffuf）
```

## 覆盖范围

### 🔍 信息收集
- 端口扫描 + 服务识别 (Nmap)
- 子域名枚举（字典+证书透明度+搜索引擎）
- 目录/文件爆破（Gobuster/ffuf）
- Web 指纹识别 / OSINT

### 🎯 Web 漏洞（17类）
| 漏洞类型 | 包含内容 |
|---------|---------|
| SQL 注入 | Union/报错/盲注/DNS外带/堆叠/二次注入/Header注入/NoSQL |
| XSS | 反射/存储/DOM/CSP绕过/Mutation XSS |
| 命令执行 | 命令拼接/绕过/参数注入/代码执行 |
| SSRF | 内网探测/云元数据/Gopher协议/DNS重绑定/编码绕过 |
| 文件上传 | 后缀绕过/内容绕过/条件竞争/云存储利用 |
| 文件包含 | LFI/RFI/伪协议/日志投毒/Session包含 |
| XXE | 文件读取/Blind OOB/XInclude/SSRF组合 |
| 反序列化 | PHP/Java/Python/.NET/Node.js |
| 越权/逻辑 | IDOR/支付篡改/竞争条件/验证码缺陷 |
| SSTI | Jinja2/FreeMarker/Velocity/Smarty/Twig/ERB |
| HTTP 请求走私 | CL.TE/TE.CL/H2降级 |
| Host 头注入 | 密码重置投毒/缓存投毒 |
| 缓存投毒 | 非键化输入/响应篡改 |
| CORS | Origin反射/Credentials |
| CRLF | 响应头注入/会话固定 |
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

## 交互流程

```
用户: /secskills
Claude: （正常待命）

用户: 对 https://target.com 做渗透测试
       ↓
Claude: ┌─────────────────────────┐
        │ ↑↓ 方向键选择            │
        │ ① 授权: 授权渗透/灰盒/黑盒/本人 │
        │ ② 深度: 标准/快速/深度/信息收集 │
        │ ③ 范围: 主目标/含子域名/关联资产 │
        └─────────────────────────┘
       ↓
Claude: ✅ 开始标准渗透测试 →
        Step 1: 信息收集（端口/子域名/指纹）
        Step 2: 漏洞发现（按攻击面加载检测知识）
        Step 3: 漏洞利用（构建利用链 A→B→C）
        Step 4: 后渗透（按需触发）
```

## 安全红线

- 🚫 无授权不输出武器化 Payload
- 🚫 禁止在生产环境执行破坏性操作
- 🚫 PoC 仅做验证，读取少量数据证明可行性
- ✅ 报告中所有敏感数据必须脱敏
- ✅ 高危漏洞立即停止深入利用，先与客户确认

## 适用场景

| 场景 | 支持 |
|------|------|
| Web 应用渗透测试 | ✅ |
| API 安全测试 | ✅ |
| 内网渗透 | ✅ |
| 移动端安全测试 | ✅ |
| 源码审计 | ✅ |
| CTF 竞赛 | ✅ |
| 红队演练 | ✅ |

## 依赖

- [Claude Code](https://claude.ai/code)（CLI 或 IDE 插件）
- 无额外工具依赖（Claude 会自动调用 Bash/WebSearch 等内置工具）

## 声明

本技能仅供合法授权的安全测试使用。使用者应确保获得目标系统所有者的书面授权。作者不对任何非法使用行为承担责任。

## License

MIT © 2025 Arenbai

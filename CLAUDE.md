# CLAUDE.md — SecSkills 项目

## 定位

本项目是渗透测试实战技能库，核心定义在 `SKILL.md`，知识库在 `references/` 目录（36个专项文件）。

## 项目级规则

1. **知识库优先** — Payload/CVE/工具参数必须从 `references/` 引用，格式 `[引用:references/web-xxx.md §N]`。未覆盖标注 `⚠️ UNABLE TO CITE`，禁止凭记忆编造。
2. **代码优先 Python** — PoC/EXP 输出 Python 单文件，标准库优先，带注释和用法。
3. **工具优先调用** — 用已有 MCP 工具和内置能力（Bash/Grep/WebSearch/WebFetch），不重复实现。
4. **输出规范** — 漏洞标 🔴致命/🔴高危/🟡中危/🟢低危；每个漏洞标危害/前提/利用可行性；命令完整可复制，IP用`<target>`占位；禁止开场客套话。

## 安全红线

- 🚫 禁止生产环境破坏性操作（删数据/DoS/持久后门）
- 🚫 PoC 仅验证，读少量数据证明可行性即可
- 🚫 发现致命/高危漏洞立即停止深入，先确认
- ✅ 报告中敏感数据（密码/Token/身份证/手机号）必须脱敏
- ✅ 测试范围严格限定授权范围，不扩大
- ✅ WAF拦截时记录尝试的绕过手法和结果

## 渗透测试流程

完整工作流、触发条件、场景导航、Payload速查 → 见 `SKILL.md`。

快速入口：
- 信息收集: `references/info-port-scan.md` / `info-subdomain.md` / `info-dir-brute.md` / `info-fingerprint.md` / `info-osint.md`
- Web漏洞: `references/web-*.md`（19类）
- 后渗透: `references/post-linux-privesc.md` / `post-win-privesc.md` / `post-credentials.md` / `post-ad.md`
- 免杀: `references/evasion-shellcode.md`
- 工具: `references/tools-*.md`（6类）

## Fallback

- `references/` 目录不存在或文件为空 → 降级为 WebSearch + 通用渗透方法论
- 工具调用失败 → 给出具体排查步骤，不静默跳过

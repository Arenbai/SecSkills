# 🔐 SecSkills — 渗透测试实战技能

> 覆盖信息收集 → 全类漏洞发现 → 漏洞利用 → 后渗透 → 免杀全流程。
> 只报告有实际危害证据的漏洞；明文传输、CORS、响应头缺失等无危害噪音项默认不报（判定门槛见 `references/web-low-value.md`）。

## 使用

在对话中直接给出目标：`对 https://target.com 做渗透测试`

## 结构

- `SKILL.md` — 主入口：触发条件、行为准则、漏洞过滤四道关卡、两阶段工作流、报告模板、场景导航索引
- `references/` — 32 个专项参考，由导航索引按需加载
  - 信息收集 ×5 / Web漏洞检测 ×14（含 web-low-value.md 低价值合并文件、web-injection-ext.md 扩展注入）/ WAF绕过 ×1
  - 主机与后渗透 ×5 / 免杀 ×1 / 工具速查 ×6

详细规则以 `SKILL.md` 为准。

License: MIT

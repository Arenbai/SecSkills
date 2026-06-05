# Web 指纹识别实战参考

> 阶段分离: [攻击] CMS/WAF/CDN/框架识别 → [利用] 历史漏洞匹配+验证 → `SKILL.md §Step 2a`

> **last_updated**: 2026-06-04 | **tested_against**: Wappalyzer, WhatWeb, httpx, Nmap NSE
---

# 攻击阶段 — 指纹采集

## 1. 自动化工具

```bash
# === WhatWeb (Ruby, 1400+插件) ===
whatweb https://target.com
whatweb -a 3 https://target.com          # 激进模式
whatweb --list-plugins | grep wordpress   # 找特定插件

# === Wappalyzer (CLI版) ===
wappalyzer https://target.com

# === webanalyze ===
webanalyze -host target.com -crawl 2

# === Finger.Fox (Httpx自带) ===
httpx -u https://target.com -tech-detect
```

## 2. 手工指纹识别

### 2.1 通过响应头

```bash
curl -I https://target.com

# Server头:
Server: Apache/2.4.41 (Ubuntu)       → Apache 2.4.41
Server: nginx/1.18.0                   → Nginx
Server: Microsoft-IIS/10.0            → IIS 10
X-Powered-By: PHP/7.4.33              → PHP 7.4.33
X-Powered-By: ASP.NET                  → ASP.NET
X-Generator: WordPress 6.3.1           → WordPress
Set-Cookie: JSESSIONID=...             → Java (Tomcat/JBoss)
Set-Cookie: PHPSESSID=...              → PHP
Set-Cookie: ASP.NET_SessionId=...      → ASP.NET
```

### 2.2 通过路径特征

```bash
# === CMS ===
/wp-admin /wp-login.php /wp-content/   → WordPress
/user/login /admin/ /sites/default/     → Drupal
/administrator/ /components/            → Joomla!
/dede/ /plus/ /templets/               → DedeCMS
/wp-admin/ /skin/                      → Z-Blog

# === 框架 ===
/static/admin/ /media/                 → Django Admin
/admin/login/?next=/                   → Django
/assets/ /bundles/                      → Symfony
/public/index.php                       → Laravel (public目录)

# === 中间件/管理 ===
/server-status /server-info            → Apache
/nginx_status                          → Nginx Status
/phpmyadmin/ /phpMyAdmin/              → phpMyAdmin
/manager/html /host-manager            → Tomcat
/solr/ /solr/admin/                    → Solr
```

### 2.3 通过JS/CSS路径

```bash
/wp-content/themes/<THEME_NAME>/       → WordPress主题
/sites/all/modules/<MODULE>/           → Drupal模块
/templates/<TEMPLATE>/css/             → Joomla模板
/content/themes/<THEME>/               → 国产CMS
```

### 2.4 通过Favicon Hash

```bash
# 获取favicon hash
curl -s https://target.com/favicon.ico | python3 -c "
import mmh3, sys, codecs
data = sys.stdin.buffer.read()
print(mmh3.hash(codecs.encode(data,'base64')))
"

# 或:
python3 -c "import requests; import mmh3; import codecs; \
  r=requests.get('https://target.com/favicon.ico'); \
  fav=codecs.encode(r.content,'base64'); \
  print(mmh3.hash(fav))"

# https://wiki.shodan.io/host-favicon-hashes
# → Shodan搜索 hash:XXXXX → 看哪些产品用这个favicon
```

---

# 利用阶段 — 版本匹配

## 3. 根据指纹找 CVE

```bash
# 确认指纹后:
searchsploit apache 2.4.41
searchsploit nginx 1.18
searchsploit wordpress 6.3.1

# 或用在线CVE库:
# https://nvd.nist.gov/
# https://cve.mitre.org/
```

## 4. 常见误报识别

```
nginx 反代 Apache → Server头可能是Apache, 实际前面是nginx
CDN加速 → IP可能不是真实IP
负载均衡 → 不同请求可能打到不同服务器(不同版本)
WAF拦截 → 可能显示假Server头
```
---

## 历史漏洞匹配检测（指纹→搜历史漏洞→验证）

> **核心价值**: 指纹识别到具体组件后，历史漏洞匹配是**投入产出比最高**的攻击路径。
> 一条准确命中的已知漏洞 ≈ 10 条通用模糊测试的结果。

> **"历史漏洞"不止 CVE**：任何公开过的漏洞都在范围内 —— CVE 编号漏洞、无编号的公开 PoC、Exploit-DB/GitHub 上的利用、厂商安全公告、框架已知缺陷（如 Shiro-550、ThinkPHP 5.x RCE）。

### 匹配流程

```
指纹结果                       多维搜索                                      PoC验证
Apache 2.4.49    →    CVE + RCE + exploit-db + github     →    实际发PoC验证
Struts 2.x       →    公开PoC + 反序列化 + 框架已知缺陷      →    OGNL注入验证
ThinkPHP         →    多版本RCE PoC + github exploit       →    RCE验证
Shiro (无版本)    →    RememberMe检测 + 反序列化特征         →    直接特征验证
WebLogic         →    T3/IIOP反序列化 + 控制台RCE           →    JNDI注入验证
```

### 搜索策略（多维度并行，优先级从高到低）

```bash
# 1. CVE 编号漏洞（有版本号优先）
WebSearch: "<组件名> <版本号> CVE exploit"

# 2. 无 CVE 的公开漏洞（中文/英文都要搜）
WebSearch: "<组件名> <版本号> RCE 漏洞 PoC"
WebSearch: "<组件名> 漏洞利用"                # 中文搜索

# 3. 特定漏洞类型（根据组件特征）
WebSearch: "<组件名> 反序列化 漏洞"
WebSearch: "<组件名> 文件上传 漏洞"

# 4. 公开利用库
WebSearch: "site:exploit-db.com <组件名>"
WebSearch: "site:github.com <组件名> exploit PoC"

# 5. 厂商/框架安全公告
WebSearch: "<组件名> security advisory <年份>"
WebSearch: "<组件名> 安全公告 漏洞"
```

### 无版本号怎么办？

> ⚠️ **有指纹无版本号 ≠ 跳过**。很多漏洞检测不依赖精确版本号。

| 组件 | 无版本号时的检测方法 |
|------|-------------------|
| **Shiro** | 直接检测 RememberMe Cookie 反序列化特征，无需版本 |
| **WebLogic** | 直接检测 T3/IIOP 协议 + 常见 RCE 链，无需版本 |
| **JBoss/WildFly** | 直接检测 /jmx-console、/web-console 等路径 + 未授权访问 |
| **Struts2** | 直接发送 OGNL 注入 Payload 检测，不依赖版本号 |
| **Tomcat** | 检测默认路径 + PUT 上传 + CVE 通用 Payload |
| **Fastjson** | 直接发送 DNSLog Payload 检测，不依赖版本 |
| **Log4j** | 直接发送 JNDI Payload（`${jndi:ldap://...}`），不依赖版本 |

### 哪些组件的指纹必须触发历史漏洞搜索

| 优先级 | 组件类型 | 示例（版本号 + 无版本号都要搜） |
|--------|---------|------|
| 🔴 必须 | Web 服务器（Apache/Nginx/IIS/Tomcat/Jetty）含具体版本 | Apache 2.4.49 → 路径穿越 RCE |
| 🔴 必须 | Java 组件（Shiro/WebLogic/JBoss/Fastjson/Log4j/Spring）| Shiro → RememberMe；Fastjson → JNDI |
| 🔴 必须 | Java 框架（Struts2/Spring MVC） | Struts2 → OGNL 注入系列 |
| 🔴 必须 | PHP 框架/CMS（ThinkPHP/Laravel/WordPress/Drupal/Discuz） | ThinkPHP → 多版本 RCE |
| 🔴 必须 | 远程管理/运维（Jenkins/GitLab/Nexus/Confluence/Jira） | Jenkins → 未授权RCE |
| 🟡 建议 | 中间件（Redis/Memcached/ES/MongoDB/RabbitMQ） | Redis → 未授权写入SSH Key |
| 🟡 建议 | 网络设备（VPN/路由器/防火墙/交换机） | FortiOS → SSL VPN RCE |
| 🟢 可选 | 前端框架（React/Vue/Angular） | 通常无远程利用，跳过 |

### 验证铁律

> ⚠️ **版本匹配 / 组件存在 ≠ 漏洞存在**。必须实际发 PoC 验证，拿到危害证据才算。

- ✅ **正确**: 指纹 → 多维度搜索 → WebFetch 获取 PoC → **实际发送请求** → 验证危害 → 过自检门
- ❌ **错误**: 看到 `Apache/2.4.49` → 直接报告 "CVE-2021-41773" — 未验证
- ❌ **错误**: 看到 `Shiro` → 报 "Shiro-550" — 版本都不确定，必须实际发送 RememberMe Payload 验证
- ❌ **错误**: 凭记忆编造 CVE 编号、版本范围、PoC — 必须从搜索结果引用

### 验证失败处理

| 失败原因 | 处理方式 |
|---------|---------|
| 漏洞已修复（回显/行为不匹配） | 跳过，不输出 |
| 版本不适用（不同分支/发行版） | 跳过，不输出 |
| PoC 条件不满足（需认证/内网/特定配置） | 跳过，标注前提条件不满足 |
| WAF/IPS 拦截 | 尝试绕过后仍失败 → 跳过 |
| 无公开 PoC | 跳过，不做猜测性报告 |

---

*参考: WhatWeb + Wappalyzer + 实战经验 | 历史漏洞匹配: SKILL.md §Step 2a*

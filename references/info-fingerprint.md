# Web 指纹识别实战参考

> 阶段分离: [攻击] CMS/WAF/CDN/框架识别 → [利用] 针对性漏洞匹配

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

*参考: WhatWeb + Wappalyzer + 实战经验*

# OSINT 信息收集实战参考

> 阶段分离: [攻击] 搜索引擎/Shodan/FOFA/Whois/邮箱 → [利用] 目标画像/凭证泄露/社会工程

---

# 攻击阶段 — 被动信息收集

## 1. Google Hacking

```bash
# 基础语法:
site:target.com                       # 全站索引
site:target.com filetype:pdf          # PDF文件(常含内部信息)
site:target.com filetype:sql          # SQL备份泄露!
site:target.com filetype:env          # .env文件泄露
site:target.com inurl:admin           # 管理后台
site:target.com inurl:login           # 登录页
site:target.com intitle:"index of"    # 目录浏览
site:target.com intext:"password"     # 页面含password
site:target.com ext:log               # 日志文件
site:target.com ext:bak               # 备份文件
site:target.com ext:conf              # 配置文件
site:target.com ext:env               # 环境变量

# 子域名+排除:
site:*.target.com -www -mail          # 子域名(排除www和mail)

# 代码仓库:
site:github.com target.com password
site:gitlab.com target.com secret
site:pastebin.com target.com

# 找员工:
site:linkedin.com "target.com"
```

## 2. Shodan

```bash
# === 基础搜索 ===
hostname:target.com                   # 主机名
org:"Target Inc"                      # 组织
ssl:target.com                         # SSL证书

# === 服务搜索 ===
port:22 target.com                    # SSH
port:3306 target.com                  # MySQL
port:9200 target.com                  # Elasticsearch
product:nginx target.com              # Nginx
product:"Apache httpd" target.com     # Apache

# === 高级 ===
city:"Beijing" port:3389              # 某城市RDP
country:CN product:"MySQL"            # 中国MySQL
vuln:CVE-2021-41773                   # 含特定CVE

# === CLI ===
shodan domain target.com
shodan host 1.2.3.4
shodan search 'hostname:target.com' --fields ip_str,port,org
```

## 3. FOFA

```bash
# === URL: https://fofa.info/ ===
domain="target.com"
host="target.com"
cert="target.com"
title="管理后台" && region="CN"
body="phpinfo()" && country="CN"
icon_hash="XXXXX"
port="6379" && country="CN" && protocol="redis"
```

## 4. Whois / DNS 历史

```bash
# Whois
whois target.com

# DNS历史 (查既往解析过的IP → 可能绕过CDN)
# https://securitytrails.com/
# https://viewdns.info/
# https://dnsdumpster.com/

# 历史页面
# https://web.archive.org/web/*/target.com
# → waybackurls
waybackurls target.com | sort -u > wayback_urls.txt
```

## 5. 邮箱与人员

```bash
# === Hunter.io ===
# 查询 target.com 的邮箱模式
# https://hunter.io/search/target.com

# === theHarvester ===
theHarvester -d target.com -b google,linkedin,crtsh,hunter

# === Phonebook ===
# https://phonebook.cz/

# === 邮箱验证 ===
# 格式: first.last@target.com → 尝试登录 → 看"用户不存在" vs "密码错误"
```

## 6. 代码仓库

```bash
# === GitHub ===
# https://github.com/search?q=target.com+password
# https://github.com/search?q=target.com+secret
# https://github.com/search?q=target.com+api_key

# truffleHog (自动扫描commit历史中的secret)
truffleHog https://github.com/target/repo.git

# gitLeaks
gitleaks detect -s https://github.com/target/repo.git

# === GitLab / Gitee ===
# 国内公司常用Gitee → 别忘了
```

## 7. SSL证书

```bash
# crt.sh
curl -s "https://crt.sh/?q=%25.target.com&output=json" | jq -r '.[].name_value'

# SSL证书中的组织信息:
openssl s_client -connect target.com:443 | openssl x509 -noout -text
# → Organization / SAN (Subject Alternative Name) → 子域名!
```

---

*参考: Google Hacking + Shodan + OSINT Framework*

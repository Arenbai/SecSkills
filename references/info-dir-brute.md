# 目录/文件爆破实战参考

> 阶段分离: [攻击] 字典生成+爆破策略 → [利用] 发现敏感文件/备份/管理后台

---

# 攻击阶段 — 字典与策略

## 1. 工具速查

```bash
# === Gobuster ===
gobuster dir -u https://target.com -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,asp,aspx,jsp,html,bak -t 50

# === ffuf (更快,更灵活) ===
ffuf -u https://target.com/FUZZ -w wordlist.txt -t 100

# === dirsearch ===
python3 dirsearch.py -u https://target.com -e php,asp,aspx,jsp,html,bak,conf,txt -t 50

# === feroxbuster (Rust, 最快) ===
feroxbuster -u https://target.com -w wordlist.txt -x php,html,bak -t 100

# === wfuzz ===
wfuzz -c -w wordlist.txt --hc 404 https://target.com/FUZZ
```

## 2. 常用字典

| 字典 | 大小 | 用途 |
|------|------|------|
| `directory-list-2.3-medium.txt` | ~220k | 通用目录 (默认) |
| `common.txt` | ~4.6k | 快速扫描 |
| `big.txt` | ~20k | 标准扫描 |
| `raft-large-directories.txt` | ~62k | 大量目录 |
| `raft-large-files.txt` | ~37k | 大量文件 |
| `SecLists/Discovery/Web-Content/` | 各种 | 场景化字典 |

## 3. 扩展名策略

```bash
# === 常见扩展名 ===
-x php,asp,aspx,jsp,do,action,phtml,phps,php5,shtml,html,htm,bak,old,zip,tar.gz,sql,txt,conf,config,ini,cfg,log,xml,json,yml,yaml,env,bkp

# === 备份文件检测 ===
# index.php → index.php.bak / index.php~ / index.php.old / index.php.save
# web.config → web.config.bak
# .htaccess → .htaccess.bak

# === Git 泄露 ===
curl https://target.com/.git/HEAD
# → 用 GitHacker / git-dumper
git-dumper https://target.com/.git/ ./output/

# === DS_Store ===
# 每个目录试: /.DS_Store → 用 ds_store 解析
```

---

# 利用阶段 — 敏感文件发现

## 4. 高价值文件清单

```bash
# === 源码备份 ===
index.php.bak / index.php~ / index.php.old
index.php.swp (vim: .index.php.swp)
www.zip / www.tar.gz / web.tar.gz / backup.zip
site.tar.gz / code.zip / backup_db.sql

# === 配置文件 ===
.env / .env.bak / .env.production
config.php / config.php.bak
web.config / web.config.bak
application.properties / application.yml
settings.py / settings.ini
database.yml / database.json
.env.local / .env.development

# === 凭证泄露 ===
.git-credentials / .npmrc / .dockercfg
AWS.config / .aws/credentials
id_rsa / id_ed25519

# === 调试页面 ===
phpinfo.php / info.php / test.php
/actuator / /actuator/env
/debug / /console / /debug/default/view
/druid/index.html (Alibaba)
/swagger-ui.html / /api-docs

# === 管理后台 ===
/admin / /administrator / /manager
/login / /wp-admin / /user/login
/dashboard / /panel / /cp
/cms / /system / /control
/editor / /phpmyadmin / /adminer
```

## 5. API 端点爆破

```bash
# === API 字典 ===
/api/v1/users /api/v2/users
/api/user/profile /api/admin/config
/api/auth/login /api/register

# === Swagger/OpenAPI ===
/swagger.json /v2/api-docs /v3/api-docs
/swagger-ui.html /openapi.json

# === GraphQL ===
/graphql /graphiql /api/graphql /query

# === Spring Boot Actuator ===
/actuator /actuator/mappings /actuator/env /actuator/heapdump
```

---

*参考: SecLists + OWASP Directory Traversal + 实战*

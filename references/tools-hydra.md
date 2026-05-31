# Hydra 速查手册

---

## 基础语法

```bash
hydra [options] <service>://<target>
hydra -L users.txt -P pass.txt <service>://<target>
hydra -l <user> -P pass.txt <service>://<target>
```

## 关键参数

| 参数 | 说明 |
|------|------|
| `-l <user>` | 单用户名 |
| `-L <file>` | 用户名字典 |
| `-p <pass>` | 单密码 |
| `-P <file>` | 密码字典 |
| `-t <n>` | 线程数 (SSH建议4) |
| `-w <n>` | 超时等待 |
| `-V` | 显示每次尝试 |
| `-f` | 找到一个就停 |
| `-o <file>` | 结果输出 |
| `-R` | 恢复上次 |
| `-e ns` | 空密码(n)+同用户名(s) |
| `-s <port>` | 自定义端口 |
| `-M <file>` | 批量目标 |

## 常用协议

```bash
# === SSH ===
hydra -l root -P pass.txt ssh://<target>
hydra -L users.txt -P pass.txt ssh://<target> -t 4

# === FTP ===
hydra -l anonymous -P pass.txt ftp://<target>
hydra -L users.txt -P pass.txt ftp://<target>

# === HTTP POST ===
hydra -l admin -P pass.txt <target> http-post-form "/login:user=^USER^&pass=^PASS^:Invalid"
hydra -l admin -P pass.txt <target> http-post-form "/login:user=^USER^&pass=^PASS^:F=Login failed"

# === HTTP Basic Auth ===
hydra -l admin -P pass.txt <target> http-get /protected/

# === HTTP GET ===
hydra -l admin -P pass.txt <target> http-get "/admin/"

# === MySQL ===
hydra -l root -P pass.txt mysql://<target>

# === MSSQL ===
hydra -l sa -P pass.txt mssql://<target>

# === RDP ===
hydra -l administrator -P pass.txt rdp://<target>

# === SMB ===
hydra -l administrator -P pass.txt smb://<target>

# === Telnet ===
hydra -l root -P pass.txt telnet://<target>

# === PostgreSQL ===
hydra -l postgres -P pass.txt postgres://<target>

# === IMAP/POP3 ===
hydra -l user@domain.com -P pass.txt imap://<target>
hydra -l user@domain.com -P pass.txt pop3://<target>

# === Redis ===
hydra -P pass.txt redis://<target>

# === VNC ===
hydra -P pass.txt vnc://<target>

# === SNMP (Community string) ===
hydra -P community.txt <target> snmp
```

## 实战技巧

```bash
# 1. 先用小字典快扫
hydra -l admin -P common_passwords.txt ssh://<target> -t 4 -f

# 2. 密码喷洒 (多用户+少密码)
hydra -L all_users.txt -p 'Spring2024!' smb://<target> -t 4

# 3. 恢复中断的session
hydra -R

# 4. 带代理
HYDRA_PROXY_HTTP=http://127.0.0.1:8080 hydra ...
```

---

*参考: Hydra 官方文档*

# 端口扫描与服务识别实战参考

> 覆盖: Nmap 核心参数 → 特殊扫描 → 防火墙规避 → 服务识别 → 脚本扫描 → 快速扫描

---

## 1. Nmap 核心命令

### 1.1 按速度分级

```bash
# === 超快速 (1-2分钟, 仅端口) ===
nmap -sS -p- --min-rate 10000 -T4 <target>           # 全端口SYN扫描
masscan -p1-65535 --rate=10000 <target>               # masscan 更快
rustscan -a <target> --range 1-65535 --batch-size 5000 # rustscan 最快

# === 快速 (5-10分钟, 端口+服务) ===
nmap -sS -sV -p 21,22,23,25,53,80,110,111,135,139,143,443,445,993,995,1723,3306,3389,5432,5900,6379,8080,8443,9001,27017,50000 -T4 <target>

# === 标准 (20-30分钟, 全面) ===
nmap -sS -sV -sC -p- -T4 -oA nmap_full <target>

# === 深入 (1-2小时, 含脚本) ===
nmap -sS -sV -sC -O -p- -T4 --script=default,safe,vuln -oA nmap_deep <target>
```

### 1.2 参数速查

| 参数 | 含义 |
|------|------|
| `-sS` | TCP SYN 扫描 (默认, 需root) |
| `-sT` | TCP Connect 扫描 (无需root) |
| `-sU` | UDP 扫描 |
| `-sV` | 服务版本检测 |
| `-sC` | 默认脚本扫描 |
| `-O` | OS 检测 |
| `-p-` | 全部 65535 端口 |
| `-p 80,443,8080` | 指定端口 |
| `--top-ports 1000` | Top 1000 端口 |
| `-T4` | 速度模板 (0-5, 4=快) |
| `--min-rate 1000` | 最小发包速率 |
| `--max-retries 2` | 最大重试 |
| `-oA name` | 输出所有格式 |
| `-oN name.nmap` | 正常文本输出 |
| `-oX name.xml` | XML 输出 |
| `-Pn` | 跳过主机发现 (禁ping) |
| `-n` | 禁止 DNS 解析 |
| `-v / -vv` | 详细/非常详细 |

---

## 2. 防火墙/IDS 规避

### 2.1 常见规避技术

```bash
# === 分片扫描 (-f) ===
nmap -sS -f <target>                          # 8字节碎片
nmap -sS -f --mtu 24 <target>                 # 指定 MTU
nmap -sS -ff <target>                         # 16字节碎片

# === 诱饵扫描 (-D) ===
nmap -sS -D RND:5 <target>                    # 5个随机诱饵IP
nmap -sS -D 192.168.1.1,10.0.0.1 <target>    # 指定诱饵

# === 源端口欺骗 ===
nmap -sS --source-port 53 <target>            # 伪装DNS流量
nmap -sS -g 80 <target>                       # 伪装HTTP流量

# === 空闲扫描 (Idle Scan) ===
nmap -sI <zombie_ip> <target>                 # 通过僵尸主机扫描

# === 延时扫描 ===
nmap -sS -T2 --scan-delay 5s <target>         # 慢速, 避免触发IDS

# === 随机顺序+假MAC ===
nmap -sS --randomize-hosts --spoof-mac Apple <target>
```

### 2.2 特殊端口扫描

```bash
# === TCP ACK 扫描 (判断防火墙规则) ===
nmap -sA <target>

# === TCP Window 扫描 ===
nmap -sW <target>

# === 自定义Flag ===
nmap -sS --scanflags SYNURG <target>
nmap -sS --scanflags URGACKPSHRSTSYNFIN <target>  # Xmas变种

# === UDP 扫描 ===
nmap -sU --top-ports 200 -T4 <target>
# UDP 慢, 建议结合:
nmap -sU -sS -p U:53,161,500,3389,T:21,22,80,443 <target>
```

---

## 3. NSE 脚本扫描

### 3.1 按类别执行

```bash
# === 漏洞扫描类 ===
nmap --script=vuln <target>                              # 全部漏洞脚本
nmap --script=smb-vuln* <target>                         # SMB漏洞
nmap --script=http-vuln* <target>                        # HTTP漏洞
nmap --script=ssl-* --script=ssl-enum-ciphers <target>   # SSL/TLS

# === 信息收集类 ===
nmap --script=discovery <target>                         # 发现类
nmap --script=http-enum,http-headers,http-title <target> # Web信息
nmap --script=smb-enum-shares,smb-os-discovery <target>  # SMB

# === 认证爆破类 ===
nmap --script=auth <target>                              # 认证
nmap --script=ftp-brute --script-args userdb=users.txt,passdb=pass.txt <target>

# === 具体漏洞检测 ===
nmap --script=http-shellshock --script-args uri=/cgi-bin/test.cgi <target>
nmap --script=http-vuln-cve2017-5638 <target>             # Struts2
nmap --script=ssl-heartbleed <target>                     # Heartbleed
```

### 3.2 常用脚本速查

| 脚本 | 用途 |
|------|------|
| `http-enum` | 目录枚举 |
| `http-title` | Web 标题 |
| `http-headers` | HTTP 头分析 |
| `http-methods` | HTTP 方法 |
| `http-shellshock` | Shellshock |
| `smb-os-discovery` | SMB OS |
| `smb-vuln-ms17-010` | 永恒之蓝 |
| `ftp-anon` | FTP 匿名登录 |
| `mysql-empty-password` | MySQL 空密码 |
| `redis-info` | Redis 信息泄露 |
| `ssl-heartbleed` | 心脏滴血 |
| `dns-zone-transfer` | DNS 域传送 |
| `snmp-info` | SNMP 信息 |
| `vnc-info` | VNC 信息 |

---

## 4. 端口中转与网络拓扑发现

### 4.1 存活主机发现

```bash
# === Ping 扫描 ===
nmap -sn 192.168.1.0/24           # ping扫描

# === ARP 扫描 (内网最快) ===
nmap -sn -PR 192.168.1.0/24
arp-scan --local                  # 专门ARP扫描

# === 无Ping端口扫描 ===
nmap -Pn -p 445 --open 192.168.1.0/24   # 扫MS17-010

# === ICMP (Ping Sweep) ===
fping -a -g 192.168.1.0/24 2>/dev/null
```

### 4.2 端口转发 / 隧道扫描

```bash
# === SSH 动态转发 (SOCKS5) ===
ssh -D 1080 user@jumphost
# proxychains nmap -sT -Pn target     # 通过 SOCKS5 扫描

# === SSH 本地转发 ===
ssh -L 4450:target:445 user@jumphost
# nmap -p 4450 localhost

# === chisel (万能隧道) ===
# Server:
./chisel server -p 8000 --reverse
# Client:
./chisel client server:8000 R:socks   # SOCKS5
```

---

## 5. 常用端口+对应攻击方向

| 端口 | 服务 | 优先检查 |
|------|------|---------|
| **21** | FTP | 匿名登录 `ftp anonymous@<target>` |
| **22** | SSH | 弱口令、私钥泄露 |
| **23** | Telnet | 弱口令 |
| **25** | SMTP | 用户枚举 `VRFY` |
| **53** | DNS | 域传送 `dig axfr @<target> domain` |
| **80/443** | HTTP/HTTPS | Web 漏洞、目录爆破 |
| **135/139/445** | SMB/RPC | MS17-010、空会话 |
| **1433** | MSSQL | 弱口令 sa |
| **1521** | Oracle | 弱口令 |
| **2049** | NFS | `showmount -e <target>` |
| **3306** | MySQL | 弱口令 root |
| **3389** | RDP | 弱口令、CVE-2019-0708 |
| **5432** | PostgreSQL | 弱口令 postgres |
| **5900** | VNC | 弱口令 |
| **6379** | Redis | 未授权访问 `redis-cli -h <target>` |
| **8080/8443/9090** | Web 管理 | Tomcat/Jenkins等 |
| **11211** | Memcached | 未授权 `echo stats \| nc <target> 11211` |
| **27017** | MongoDB | 未授权 |
| **50000** | SAP | SAP 管理 |
| **61616** | ActiveMQ | 未授权/CVE |

---

## 6. 快速扫描技巧

```bash
# === 第一波: 30秒存活+常用端口 ===
nmap -sn -T5 192.168.1.0/24 -oA quick_ping
nmap -sS -sV -p 21,22,23,25,53,80,135,139,443,445,1433,3306,3389,5432,6379,8080,8443,27017 \
  --open -iL alive_hosts.txt -T4 -oA quick_scan

# === 第二波: 深度扫描存活主机的全端口 ===
nmap -sS -sV -sC -p- -T4 --open -iL alive_hosts.txt -oA deep_scan

# === 第三波: 漏洞脚本精准扫描 ===
nmap -sS -sV --script=vuln -p $(从deep_scan提取的开放端口) -iL alive_hosts.txt -oA vuln_scan
```

---

## 7. 输出解析

```bash
# 从 nmap 提取开放端口列表
grep -E '^[0-9]+/tcp' scan.nmap | awk -F/ '{print $1}' | tr '\n' , | sed 's/,$//'

# 从多个XML文件生成表格
nmap -sS -p- <target> -oX scan.xml
# 用 python 解析:
python3 -c "
import xml.etree.ElementTree as ET
tree = ET.parse('scan.xml')
for host in tree.findall('.//host'):
    for port in host.findall('.//port'):
        print(f\"{port.attrib['portid']}/{port.attrib['protocol']} - {port.find('service').attrib.get('name','unknown')} - {port.find('service').attrib.get('product','')}\")
"
```

---

*参考: Nmap 官方文档 + 实战经验整理*

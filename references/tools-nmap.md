# Nmap 速查手册

---

## 扫描类型

| 参数 | 用途 | 场景 |
|------|------|------|
| `-sS` | TCP SYN 扫描 (半开) | ★ 默认首选, 需root |
| `-sT` | TCP Connect 扫描 | 无需root, 较慢 |
| `-sU` | UDP 扫描 | DNS/SNMP/NTP/NetBIOS |
| `-sA` | TCP ACK 扫描 | 防火墙规则探测 |
| `-sW` | TCP Window 扫描 | 更隐蔽的端口状态判断 |
| `-sN/sF/sX` | Null/FIN/Xmas | 绕过简单防火墙 |

## 端口参数

| 参数 | 示例 |
|------|------|
| `-p-` | 全端口 1-65535 |
| `-p 80,443,8080` | 指定端口 |
| `-p 1-1000` | 范围 |
| `--top-ports 100` | Top N |
| `-p U:53,T:80` | 混合 UDP/TCP |

## 服务与OS

| 参数 | 用途 |
|------|------|
| `-sV` | 服务版本检测 |
| `-sC` | 默认脚本 |
| `-O` | OS检测 |
| `-A` | 一键: -sV -sC -O -traceroute |

## 速度与隐蔽

| 参数 | 用途 |
|------|------|
| `-T0` ~ `-T5` | 速度模板 (0=偏执 5=疯狂) |
| `-T4` | ★ 推荐: 快且可靠 |
| `--min-rate 1000` | 最小发包率 |
| `--max-retries 2` | 最大重试 |
| `-f` | 分片 (8字节) |
| `--mtu 24` | 自定义MTU分片 |
| `-D RND:5` | 5个随机诱饵IP |
| `--source-port 53` | 伪装源端口 (DNS) |
| `-g 80` | 同 `--source-port` |
| `-sI zombie_ip` | Idle Scan (僵尸扫描) |

## 输出

| 参数 | 格式 |
|------|------|
| `-oN file.nmap` | 文本 |
| `-oX file.xml` | XML |
| `-oG file.gnmap` | Grepable |
| `-oA name` | 所有格式 |
| `-v / -vv` | 详细 |
| `--open` | 只显示开放端口 |
| `--reason` | 显示判断依据 |

## NSE 脚本

```bash
# 分类:
--script=auth       # 认证
--script=vuln       # ★ 漏洞检测
--script=discovery  # 发现
--script=exploit    # 利用
--script=safe       # 安全(不攻击)
--script=default    # 默认

# 常用:
--script=http-vuln*              # HTTP漏洞
--script=smb-vuln*               # SMB漏洞 (含MS17-010)
--script=ssl-*                   # SSL/TLS检查
--script=ftp-anon                # FTP匿名登录
--script=mysql-empty-password    # MySQL空密码
--script=redis-info              # Redis信息
```

## 实战组合

```bash
# 1. 快速全端口
nmap -sS -p- --min-rate 10000 -T4 --open <target>

# 2. 对开放端口做服务+脚本
nmap -sS -sV -sC -p <ports> -T4 -oA detail <target>

# 3. 漏洞扫描
nmap -sS -sV --script=vuln -p <ports> -oA vuln <target>

# 4. UDP Top 200
nmap -sU --top-ports 200 -T4 <target>
```

---

*参考: Nmap 官方文档*

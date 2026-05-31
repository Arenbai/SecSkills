# 子域名枚举实战参考

> 分类: 证书透明 → DNS → 字典爆破 → 搜索引擎 → 工具链

---

## 1. 工具链速查

### 1.1 常用工具对比

| 工具 | 速度 | 发现量 | 特点 |
|------|------|--------|------|
| **subfinder** | 快 | 中 | 被动收集 (证书+API), 不主动爆破 |
| **amass** | 慢 | 多 | 全面, 被动+主动, OWASP维护 |
| **assetfinder** | 快 | 中 | 轻量, 被动收集 |
| **findomain** | 快 | 中 | 证书透明+API |
| **shuffledns** | 中 | 多 | 纯DNS解析, 配合massdns |
| **dnsx** | 快 | — | DNS工具集, 配合其他用 |
| **chaos** | — | 多 | ProjectDiscovery的公共数据集 |

### 1.2 组合命令 (推荐)

```bash
# === 第一步: 被动收集 (不产生大流量, 快速) ===
subfinder -d target.com -o subs_passive.txt
assetfinder --subs-only target.com >> subs_passive.txt
findomain -t target.com -q >> subs_passive.txt

# 去重
sort -u subs_passive.txt -o subs_unique.txt

# === 第二步: DNS 解析 (确认存活) ===
dnsx -l subs_unique.txt -resp-only -o subs_alive.txt

# === 第三步: 主动爆破 (可选, 动静大) ===
# 需要好的字典
shuffledns -d target.com -w subdomains-top1million-5000.txt -r resolvers.txt -o subs_brute.txt

# === 第四步: 合并 + 解析 + HTTP探测 ===
cat subs_alive.txt subs_brute.txt | sort -u | dnsx -resp-only | httpx -o subs_http.txt
```

---

## 2. 证书透明 (Certificate Transparency)

### 2.1 crt.sh

```bash
# 在线查看: https://crt.sh/?q=%25.target.com

# API 方式获取:
curl -s "https://crt.sh/?q=%25.target.com&output=json" | jq -r '.[].name_value' | sed 's/\*\.//g' | sort -u

# 或一行:
curl -s "https://crt.sh/?q=%25.target.com&output=json" | jq -r '.[].name_value' | tr '\n' '\n' | sed 's/^\*\.//' | sort -u > crtsh_subs.txt
```

### 2.2 其他证书源

```bash
# Google Transparency Report
# https://transparencyreport.google.com/https/certificates

# Facebook Certificate Transparency
curl -s "https://developers.facebook.com/tools/ct/search/?query=target.com" | grep -oP '[a-zA-Z0-9.-]+\.target\.com' | sort -u

# CertSpotter
curl -s "https://api.certspotter.com/v1/issuances?domain=target.com&expand=dns_names" | jq -r '.[].dns_names[]' | sort -u
```

---

## 3. DNS 爆破

### 3.1 字典

```bash
# 推荐字典:
# 1. SecLists: https://github.com/danielmiessler/SecLists/tree/master/Discovery/DNS
# 2. Assetnote: https://wordlists-cdn.assetnote.io/data/automated.json
# 3. jhaddix all.txt: 聚合字典

# 首先用小字典 (top 5000)
shuffledns -d target.com -w subdomains-top1million-5000.txt -r resolvers.txt

# 然后针对大企业用大字典 (top 100k+)
shuffledns -d target.com -w all.txt -r resolvers.txt -o subs_dns_brute.txt
```

### 3.2 置换扫描 (Permutation)

```bash
# 对已知子域名做置换 (dev/staging/test/etc.)
# 已知: www.target.com, api.target.com
# 生成: dev.www.target.com, staging.api.target.com, www-dev.target.com...

# 工具: gotator / altdns
gotator -sub subs_alive.txt -perm permutations.txt -depth 1 -numbers 10 -o perm_subs.txt

# 然后解析:
dnsx -l perm_subs.txt -resp-only -o perm_alive.txt
```

---

## 4. DNS 区域传送 (Zone Transfer)

```bash
# === 尝试 AXFR ===
dig axfr @ns1.target.com target.com
dig axfr @dns.target.com target.com

# === 遍历所有 NS ===
for ns in $(dig +short NS target.com); do
  echo "=== $ns ==="
  dig axfr @$ns target.com
done
```

---

## 5. 搜索引擎 + 第三方

```bash
# === Google Hacking ===
site:*.target.com
site:target.com -www

# === Shodan ===
hostname:target.com
ssl:target.com
ssl.cert.subject.cn:target.com

# === FOFA ===
domain="target.com" && cert="target.com"

# === VirusTotal ===
https://www.virustotal.com/gui/domain/target.com/relations
# → subdomains 选项卡

# === SecurityTrails ===
curl -s "https://api.securitytrails.com/v1/domain/target.com/subdomains" \
  -H "APIKEY: your-key" | jq -r '.subdomains[]' | sed "s/$/.target.com/"

# === AlienVault OTX ===
curl -s "https://otx.alienvault.com/api/v1/indicators/domain/target.com/passive_dns" | jq -r '.passive_dns[].hostname' | sort -u

# === Chaos (ProjectDiscovery) ===
chaos -d target.com -o chaos_subs.txt
```

---

## 6. 子域名信息扩展

### 6.1 HTTP 探测

```bash
# 识别 Web 服务
cat subs_alive.txt | httpx -title -status-code -tech-detect -o http_info.txt

# 截图 (看长什么样的网站)
cat http_info.txt | aquatone -out aquatone_output

# 按状态码分类:
grep "200" http_info.txt > web_200.txt
grep "403" http_info.txt > web_403.txt   # 403可能是敏感后台
grep "401" http_info.txt > web_401.txt
```

### 6.2 端口扫描 (子域名)

```bash
# 对解析出的 IP 扫端口
cat subs_alive.txt | dnsx -a -resp-only | sort -u > ips.txt
nmap -sS -sV -p 21,22,80,443,8080,8443,9090 -iL ips.txt -T4 -oA nmap_subs
```

### 6.3 目录爆破 (Top 子域名)

```bash
# 优先爆破 200 OK + 有趣标题的子域名
cat web_200.txt | head -20 | while read url; do
  ffuf -u "$url/FUZZ" -w common.txt -o ffuf_$(echo $url | md5sum | cut -c1-8).json
done
```

---

## 7. 快速参考

```bash
# === 30秒快速收集 ===
subfinder -d target.com -silent
assetfinder --subs-only target.com

# === 5分钟标准收集 ===
subfinder -d target.com | anew subs.txt
assetfinder --subs-only target.com | anew subs.txt
findomain -t target.com -q | anew subs.txt
curl -s "https://crt.sh/?q=%25.target.com&output=json" | jq -r '.[].name_value' | anew subs.txt
dnsx -l subs.txt -resp-only -o subs_alive.txt
httpx -l subs_alive.txt -o subs_http.txt

# 看结果:
wc -l subs*.txt
cat subs_http.txt
```

---

*参考: Subfinder + Amass + Assetfinder + 实战*

# 模糊测试与爆破工具速查

> **last_updated**: 2026-06-04 | **tested_against**: Gobuster 3.6, ffuf 2.x, Feroxbuster 2.x
---

## Gobuster

```bash
# === 目录爆破 ===
gobuster dir -u https://target.com -w wordlist.txt -x php,html,bak -t 50

# === DNS子域名爆破 ===
gobuster dns -d target.com -w subdomains.txt

# === VHOST爆破 ===
gobuster vhost -u https://target.com -w vhosts.txt

# === S3 Bucket ===
gobuster s3 -w bucket-names.txt

# === 常用参数 ===
-t 50         # 线程数
-x php,html   # 扩展名
-k            # 跳过SSL证书验证
-o result.txt # 输出到文件
--wildcard    # 检测泛解析
```

## ffuf (fuff)

```bash
# === 目录爆破 ===
ffuf -u https://target.com/FUZZ -w wordlist.txt -t 100

# === 带扩展名 ===
ffuf -u https://target.com/FUZZ -w wordlist.txt -e .php,.html,.bak -t 100

# === 参数Fuzz ===
# GET参数
ffuf -u 'https://target.com/page.php?FUZZ=test' -w params.txt
# POST参数
ffuf -u https://target.com/login -w params.txt -X POST -d 'user=admin&FUZZ=test'

# === VHOST爆破 ===
ffuf -u https://FUZZ.target.com -w vhosts.txt

# === 按状态码/长度过滤 ===
ffuf -u URL -w wordlist.txt -fc 404             # 过滤404
ffuf -u URL -w wordlist.txt -mc 200,302         # 仅这些状态码
ffuf -u URL -w wordlist.txt -fs 1234            # 过滤特定长度

# === Cookie/Header ===
ffuf -u URL -w wordlist.txt -H "Cookie: session=xxx"
ffuf -u URL -w wordlist.txt -H "Authorization: Bearer xxx"
```

## Dirsearch

```bash
# 基础
python3 dirsearch.py -u https://target.com -e php,asp,aspx,jsp,html,bak,conf

# 深度扫描
python3 dirsearch.py -u https://target.com -e php,asp,aspx -t 30 --deep-recursive

# 走代理 (Burp)
python3 dirsearch.py -u https://target.com --proxy=http://127.0.0.1:8080
```

## Feroxbuster (Rust, 极快)

```bash
feroxbuster -u https://target.com -w wordlist.txt -x php,html,bak -t 100

# 递归扫描
feroxbuster -u https://target.com -w wordlist.txt --depth 2
```

## Wfuzz

```bash
# 目录爆破
wfuzz -c -w wordlist.txt --hc 404 https://target.com/FUZZ

# 参数爆破
wfuzz -c -w params.txt --hc 404 "https://target.com/page.php?FUZZ=test"

# 数字范围 FUZZ
wfuzz -c -z range,1-1000 "https://target.com/api/user/FUZZ"

# POST JSON
wfuzz -c -w wordlist.txt -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"FUZZ"}' "https://target.com/api/login"
```

---

*参考: 各工具官方文档*

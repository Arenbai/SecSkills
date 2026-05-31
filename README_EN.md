# SecSkills — Penetration Testing Skill for Claude Code

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-orange)](https://claude.ai/code)
[![Version](https://img.shields.io/badge/version-1.2-green.svg)]()

A comprehensive penetration testing skill pack designed for **Claude Code**. Covers the full kill chain: Recon → Vulnerability Discovery → Exploitation → Post-Exploitation → Evasion. When you tell Claude "test this site," it presents an arrow-key selection panel to confirm authorization and scope, then executes a complete penetration testing workflow.

## Quick Start

### Installation

```bash
# Clone into Claude Code skills directory
git clone https://github.com/Arenbai/SecSkills.git ~/.claude/skills/secskills

# Or into project-local .claude/skills/
git clone https://github.com/Arenbai/SecSkills.git .claude/skills/secskills
```

### Usage

Just say this in Claude Code:

```
Penetration test https://target.com
```

Claude will present an **arrow-key selection panel** (↑↓ to navigate, Enter to confirm) for authorization level, test depth, and scope — then start automatically.

## Architecture

```
SecSkills/
├── SKILL.md              # Core skill file (workflow + navigation index)
└── references/           # Knowledge base (34 specialized files)
    ├── info-*.md         # Recon (port scan/subdomain/directory/fingerprint/OSINT)
    ├── web-*.md          # Web vulns (SQLi→XSS→RCE→SSRF→Deserialization... 17 types)
    ├── post-*.md         # Post-exploitation (Linux/Windows privesc/credentials/AD)
    ├── host-*.md         # Host services (brute force)
    ├── evasion-*.md      # Evasion (shellcode obfuscation + loaders)
    └── tools-*.md        # Tool cheatsheets (nmap/sqlmap/msf/hydra/impacket/ffuf)
```

## Coverage

### 🔍 Reconnaissance
- Port scanning + service identification (Nmap)
- Subdomain enumeration (DNS + CT logs + search engines)
- Directory/file brute-forcing (Gobuster/ffuf)
- Web fingerprinting / OSINT

### 🎯 Web Vulnerabilities (17 categories)
| Category | Details |
|----------|---------|
| SQL Injection | Union/Error/Blind/OOB/Stacked/Second-order/Header/NoSQL |
| XSS | Reflected/Stored/DOM/CSP bypass/Mutation XSS |
| Command Injection | Command chaining/Bypass/Parameter injection/Code execution |
| SSRF | Internal probing/Cloud metadata/Gopher/DNS rebinding |
| File Upload | Extension bypass/Content bypass/Race condition/Cloud exploitation |
| File Inclusion | LFI/RFI/PHP wrappers/Log poisoning/Session inclusion |
| XXE | File read/Blind OOB/XInclude/SSRF chaining |
| Deserialization | PHP/Java/Python/.NET/Node.js |
| Auth/Logic | IDOR/Payment manipulation/Race condition/CAPTCHA flaws |
| SSTI | Jinja2/FreeMarker/Velocity/Smarty/Twig/ERB |
| HTTP Request Smuggling | CL.TE/TE.CL/HTTP2 downgrade |
| Host Header Injection | Password reset poisoning/Cache poisoning |
| Cache Poisoning | Unkeyed inputs/Response manipulation |
| CORS | Origin reflection/Credentials exposure |
| CRLF Injection | Response header injection/Session fixation |
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

## Interaction Flow

```
User: /secskills
Claude: (Ready, no popup)

User: Penetration test https://target.com
       ↓
Claude: ┌──────────────────────────────┐
        │ ↑↓ Arrow keys to select      │
        │ ① Auth: Red Team/Gray/Black/Own │
        │ ② Depth: Standard/Quick/Deep/Recon │
        │ ③ Scope: Target/Subdomains/Related │
        └──────────────────────────────┘
       ↓
Claude: ✅ Starting standard pentest →
        Step 1: Recon (ports/subdomains/fingerprints)
        Step 2: Discovery (load detection knowledge per attack surface)
        Step 3: Exploitation (build exploit chains A→B→C)
        Step 4: Post-exploitation (on demand)
```

## Safety Rules

- 🚫 No weaponized payloads without authorization
- 🚫 No destructive operations on production
- 🚫 PoC verification only — read minimal data to prove impact
- ✅ All sensitive data must be redacted in reports
- ✅ Stop and notify the client immediately upon discovering critical vulnerabilities

## Use Cases

| Scenario | Supported |
|----------|-----------|
| Web Application Pentest | ✅ |
| API Security Testing | ✅ |
| Internal Network Pentest | ✅ |
| Mobile Security Testing | ✅ |
| Source Code Audit | ✅ |
| CTF Competitions | ✅ |
| Red Team Exercises | ✅ |

## Requirements

- [Claude Code](https://claude.ai/code) (CLI or IDE extension)
- No external tool dependencies (Claude auto-invokes built-in tools like Bash/WebSearch)

## Disclaimer

This skill is intended for legally authorized security testing only. Users must obtain written authorization from the target system owner before use. The author assumes no responsibility for any illegal use.

## License

MIT © 2025 Arenbai

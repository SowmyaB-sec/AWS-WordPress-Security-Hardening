# 🧪 Security Testing — Methodology & Results

This document covers the full security testing procedure applied to validate
the hardening of the EC2 infrastructure, LAMP stack, and WordPress application.
Testing was conducted in two phases: before hardening to establish a baseline,
and after hardening to confirm remediation.

> ⚠️ Legal notice: All testing documented here was performed exclusively against
> infrastructure owned and operated by the author. Never run security scans
> against systems you do not own or have explicit written authorisation to test.
> Unauthorised scanning is illegal in most jurisdictions.

---

## Contents

| Section | What it covers |
|---------|---------------|
| [1. Testing Methodology](#1-testing-methodology) | Phases, scope, and approach |
| [2. Nmap — Port and Service Enumeration](#2-nmap--port-and-service-enumeration) | Pre and post hardening scans |
| [3. Nessus — Vulnerability Scanning](#3-nessus--vulnerability-scanning) | Setup, scan types, and findings |
| [4. WPScan — WordPress Vulnerability Scanner](#4-wpscan--wordpress-vulnerability-scanner) | WordPress-specific scanning |
| [5. Nikto — Web Server Scanner](#5-nikto--web-server-scanner) | Misconfiguration and header analysis |
| [6. Wireshark and tcpdump — Traffic Analysis](#6-wireshark-and-tcpdump--traffic-analysis) | Confirming encrypted traffic |
| [7. SSL Labs — TLS Configuration Audit](#7-ssl-labs--tls-configuration-audit) | Achieving A+ rating |
| [8. Mozilla Observatory — Header Analysis](#8-mozilla-observatory--header-analysis) | HTTP security header scoring |
| [9. Kali Linux — Manual Testing](#9-kali-linux--manual-testing) | Additional tools and manual checks |
| [10. Pre and Post Hardening Comparison](#10-pre-and-post-hardening-comparison) | Summary of findings before and after |
| [11. Verification Checklist](#11-verification-checklist) | Final testing sign-off |

---

## 1. Testing Methodology

### Phases

Testing followed a structured four-phase approach:

**Phase 1 — Baseline (Before Hardening)**
Document the attack surface of the default, unconfigured installation.
Every finding at this stage is expected — the purpose is to record what
an attacker would see on day one.

**Phase 2 — Hardening**
Apply all controls documented in `infrastructure/README.md`,
`lamp-stack/README.md`, and `wordpress/README.md`.

**Phase 3 — Validation (After Hardening)**
Re-run all tests from Phase 1 using identical commands and configurations.
Compare results against the baseline to confirm each finding has been
remediated or has an accepted residual risk.

**Phase 4 — Documentation**
Record all findings, remediation actions, and residual risks in the
security report at `reports/security-report.md`.

### Scope

| In scope | Out of scope |
|----------|-------------|
| EC2 instance public IP | AWS management console |
| `https://yourdomain.com` | Other AWS account resources |
| WordPress admin panel (authenticated testing) | Third-party plugin vendor systems |
| All ports on the EC2 public IP | Internal VPC resources not exposed externally |

---

## 2. Nmap — Port and Service Enumeration

Nmap is used to enumerate which ports are open and what services are running.
The before-hardening scan reveals the attack surface; the after-hardening scan
confirms it has been reduced to the minimum required.

### Run from your Kali Linux machine

### Before Hardening — Full Port Scan

```bash
# Full TCP port scan with service version detection and default scripts
nmap -sV -sC -p- --open -T4 YOUR_ELASTIC_IP \
     -oN nmap/tcp_full_before.txt

# Top 100 UDP ports
nmap -sU --top-ports 100 YOUR_ELASTIC_IP \
     -oN nmap/udp_top100_before.txt

# OS detection (requires root)
sudo nmap -O YOUR_ELASTIC_IP \
     -oN nmap/os_detect_before.txt
```

### Expected Before Hardening Output

At minimum, a default LAMP installation typically exposes:
```
22/tcp    open  ssh      OpenSSH 8.7 (protocol 2.0)
80/tcp    open  http     Apache httpd 2.4.54
443/tcp   open  ssl/http Apache httpd 2.4.54
3306/tcp  open  mysql    MySQL 8.0.30
```
In more poorly configured instances, additional ports may be open including
8080, 8443, or ephemeral ports from development tools.

### After Hardening — Full Port Scan

```bash
nmap -sV -sC -p- --open -T4 YOUR_ELASTIC_IP \
     -oN nmap/tcp_full_after.txt

nmap -sU --top-ports 100 YOUR_ELASTIC_IP \
     -oN nmap/udp_top100_after.txt
```

### Expected After Hardening Output
```
80/tcp    open  http    Apache httpd (redirects to HTTPS)
443/tcp   open  ssl/http Apache httpd
2222/tcp  open  ssh     OpenSSH (no version banner)
```
Key changes to confirm:
- Port 3306 is absent — MySQL is no longer reachable externally
- SSH is on port 2222, not 22
- SSH banner does not reveal the exact OpenSSH version
- No unexpected ports open

### Service Version Suppression Check

```bash
# Confirm Apache version is not disclosed in the banner
nmap -sV -p 443 YOUR_ELASTIC_IP
```

Before hardening: `Apache httpd 2.4.54 ((Amazon Linux))`
After hardening: `Apache httpd` (version suppressed by `ServerTokens Prod`)

---

## 3. Nessus — Vulnerability Scanning

Nessus performs a comprehensive vulnerability assessment, cross-referencing
installed software versions against the CVE database and checking for common
misconfigurations.

### Setup — Nessus Essentials (Free)

1. Download Nessus Essentials from [tenable.com/products/nessus/nessus-essentials](https://www.tenable.com/products/nessus/nessus-essentials)
2. Install on your Kali Linux machine:
```bash
   sudo dpkg -i Nessus-*-debian10_amd64.deb
   sudo systemctl start nessusd
```
3. Navigate to `https://localhost:8834` in a browser
4. Register with a free Nessus Essentials licence key
5. Wait for plugin compilation to complete (can take 30–60 minutes on first run)

### Scan Configuration — Before Hardening

1. Navigate to **New Scan → Basic Network Scan**
2. Name: `LAMP-WordPress Before Hardening`
3. Target: `YOUR_ELASTIC_IP`
4. Under **Discovery**: scan all ports
5. Under **Assessment**: Web Application Tests → enabled
6. Save and launch the scan

### Scan Configuration — After Hardening

Duplicate the scan with name `LAMP-WordPress After Hardening` and re-run
against the same target after all hardening steps are complete.

### Key Vulnerability Categories Checked

| Category | What Nessus looks for |
|----------|----------------------|
| CVE — Apache | Known vulnerabilities in the Apache version detected |
| CVE — MySQL | Known vulnerabilities in the MySQL version detected |
| CVE — PHP | Known PHP vulnerabilities and dangerous configurations |
| CVE — OpenSSH | SSH vulnerabilities and weak algorithm support |
| SSL/TLS | Weak protocols, weak ciphers, certificate issues |
| WordPress | Plugin CVEs, version disclosure, default files |
| Misconfiguration | Open redirects, default credentials, exposed admin interfaces |

### Export Reports

After each scan completes:
1. Navigate to the scan → **Report**
2. Export as **PDF** for the portfolio report
3. Save to `nessus/before_hardening_report.pdf` and `nessus/after_hardening_report.pdf`

### Expected Results

| Severity | Before hardening | After hardening |
|----------|-----------------|-----------------|
| Critical | 2–5 (varies by patch level) | 0 |
| High | 3–8 | 0 |
| Medium | 5–15 | 1–3 (accepted residuals) |
| Low | 10+ | 5–10 |
| Info | Many | Many (informational only) |

> Fill in the actual numbers from your scans — these are indicative ranges
> based on a typical default Amazon Linux 2023 + LAMP installation.

---

## 4. WPScan — WordPress Vulnerability Scanner

WPScan is purpose-built for WordPress and checks for vulnerabilities that
generic scanners like Nessus do not cover in depth — outdated plugins,
user enumeration, credential exposure, and WordPress-specific misconfigurations.

### Install WPScan on Kali Linux

WPScan is pre-installed on Kali Linux. Update it first:

```bash
sudo gem update wpscan
```

Or install from scratch:

```bash
sudo gem install wpscan
```

### Obtain a Free WPScan API Token

Register at [wpscan.com](https://wpscan.com) — the free tier provides 25 API
requests per day, which is sufficient for this project. The API token enables
CVE lookups against the WPScan vulnerability database.

### Run WPScan — Before Hardening

```bash
wpscan --url https://yourdomain.com \
       --api-token YOUR_WPSCAN_API_TOKEN \
       --enumerate u,vp,vt,tt,cb,dbe,m \
       --plugins-detection aggressive \
       -o testing/wpscan/before_hardening.txt \
       --format cli-no-colour
```

Flag explanation:
- `u` — enumerate usernames
- `vp` — vulnerable plugins
- `vt` — vulnerable themes
- `tt` — timthumbs (old vulnerable resize script)
- `cb` — config backups
- `dbe` — database exports
- `m` — media IDs
- `--plugins-detection aggressive` — more thorough but slower

### Run WPScan — After Hardening

```bash
wpscan --url https://yourdomain.com \
       --api-token YOUR_WPSCAN_API_TOKEN \
       --enumerate u,vp,vt,cb,dbe \
       -o testing/wpscan/after_hardening.txt \
       --format cli-no-colour
```

### What to Look For in Results

| Finding | Before hardening | After hardening |
|---------|-----------------|-----------------|
| WordPress version exposed | Yes — visible in page source | No — removed from output |
| Username enumeration | `admin` discoverable via `?author=1` | Blocked — returns 403 |
| XML-RPC enabled | Yes | No — returns 403 |
| `readme.html` accessible | Yes — reveals WP version | No — returns 403 |
| Vulnerable plugins | Possibly — if not updated | 0 — auto-updates active |
| Config backup files | Check for `wp-config.php.bak` etc | None accessible |
| `/wp-admin/` accessible | Yes | No — returns 404 |

---

## 5. Nikto — Web Server Scanner

Nikto scans web servers for known misconfigurations, outdated software
signatures, dangerous default files, and missing security headers.

### Install on Kali Linux

Pre-installed on Kali Linux. Update first:

```bash
sudo apt update && sudo apt install nikto -y
```

### Run Nikto — Before Hardening

```bash
nikto -h https://yourdomain.com \
      -ssl \
      -port 443 \
      -output testing/nikto/before_hardening.txt \
      -Format txt
```

### Run Nikto — After Hardening

```bash
nikto -h https://yourdomain.com \
      -ssl \
      -port 443 \
      -output testing/nikto/after_hardening.txt \
      -Format txt
```

### Key Findings Nikto Checks

| Check | Before hardening | After hardening |
|-------|-----------------|-----------------|
| Server version header | Apache version disclosed | `Server: Apache` only |
| X-Frame-Options | Missing | Present: SAMEORIGIN |
| X-Content-Type-Options | Missing | Present: nosniff |
| X-XSS-Protection | Missing | Present: 1; mode=block |
| TRACE method enabled | Often enabled by default | Disabled |
| `/wp-login.php` accessible | Yes | No — 404 |
| `/xmlrpc.php` accessible | Yes | No — 403 |
| `/readme.html` accessible | Yes | No — 403 |
| Directory indexing | Potentially enabled | Disabled |
| Default Apache test page | May be present | Replaced with WordPress |

---

## 6. Wireshark and tcpdump — Traffic Analysis

Traffic analysis confirms that all web traffic is encrypted and that no
credentials or sensitive data are transmitted in plaintext.

### Capture Traffic on the EC2 Instance with tcpdump

Run on the EC2 instance while performing a WordPress login from your browser:

```bash
# Capture all traffic on port 80 to confirm HTTP → HTTPS redirect
# and that no credentials appear in plaintext
sudo tcpdump -i eth0 -n port 80 -w /tmp/http_capture.pcap

# Capture HTTPS traffic to confirm encryption (you should see TLS records,
# not plaintext HTTP)
sudo tcpdump -i eth0 -n port 443 -w /tmp/https_capture.pcap

# Capture for 60 seconds then stop
sudo tcpdump -i eth0 -G 60 -W 1 -w /tmp/full_capture.pcap
```

Transfer the `.pcap` files to your local machine for analysis in Wireshark:

```bash
scp -i YOUR_KEY.pem -P 2222 \
    ec2-user@YOUR_ELASTIC_IP:/tmp/*.pcap \
    ./testing/wireshark/
```

### Analyse in Wireshark

Open each file in Wireshark and check:

**HTTP capture (port 80):**
- All requests should show HTTP 301 redirect responses
- No request body content should be visible
- No credentials should appear in any captured packet

**HTTPS capture (port 443):**
- Traffic should show TLS handshake records (`Client Hello`, `Server Hello`,
  `Certificate`, `Application Data`)
- `Application Data` records should be opaque — you cannot read the content
  (this confirms encryption is working)
- No plaintext HTTP content should be visible

**What a problem looks like:**
- If you can read WordPress login credentials in the HTTP capture, HTTPS
  redirect is not working
- If you see plaintext content in the port 443 capture, SSL termination
  is broken

### Wireshark Filter Commands for Analysis
- **Show only HTTP traffic** : http
- **Show only TLS traffic**: tls
- **Show only packets containing the word "password" (should return nothing on HTTPS)**: frame contains "password"
- **Show TCP packets with RST flags (connection resets — may indicate blocked scans)**: tcp.flags.reset == 1

- ---

## 7. SSL Labs — TLS Configuration Audit

SSL Labs performs a comprehensive analysis of the TLS configuration and
grades it on a scale from A+ to F.

### Run the Test

Visit: `https://www.ssllabs.com/ssltest/analyze.html?d=yourdomain.com`

Allow 2–5 minutes for the full analysis to complete.

### Target Grade: A+

Requirements for A+:

| Requirement | Configuration |
|-------------|--------------|
| TLS 1.3 supported | Enabled in `ssl.conf` |
| TLS 1.2 supported | Enabled in `ssl.conf` |
| TLS 1.0 and 1.1 disabled | `SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1` |
| Weak ciphers absent | Strong cipher suite list in `ssl.conf` |
| HSTS enabled | `Strict-Transport-Security` header with `max-age=31536000` |
| HSTS preload eligible | Header includes `preload; includeSubDomains` |
| Forward secrecy | ECDHE cipher suites prioritised |
| No certificate chain issues | Let's Encrypt full chain installed |
| OCSP stapling | `SSLUseStapling on` in `ssl.conf` |

### Common Reasons for Less Than A+

| Grade drop | Cause | Fix |
|-----------|-------|-----|
| A instead of A+ | HSTS max-age too short or preload missing | Set `max-age=31536000; includeSubDomains; preload` |
| B | TLS 1.0 or 1.1 still enabled | Add `-TLSv1 -TLSv1.1` to `SSLProtocol` |
| B | Weak cipher suites | Update `SSLCipherSuite` to strong list |
| C | Self-signed or expired certificate | Install valid Let's Encrypt certificate |
| F | TLS not configured | Install `mod_ssl` and configure HTTPS VirtualHost |

---

## 8. Mozilla Observatory — Header Analysis

Mozilla Observatory analyses HTTP security headers and grades the configuration.

### Run the Test

Visit: `https://observatory.mozilla.org` and enter `yourdomain.com`

### Target Score: A+ (100+)

Headers required for maximum score:

| Header | Required value | Set in |
|--------|---------------|--------|
| Content-Security-Policy | Restrictive policy | `security.conf` |
| X-Frame-Options | `SAMEORIGIN` or `DENY` | `security.conf` |
| X-Content-Type-Options | `nosniff` | `security.conf` |
| Strict-Transport-Security | `max-age=31536000; includeSubDomains; preload` | `security.conf` |
| Referrer-Policy | `strict-origin-when-cross-origin` or stricter | `security.conf` |
| Permissions-Policy | Restrictive policy | `security.conf` |
| X-XSS-Protection | `1; mode=block` (legacy browsers) | `security.conf` |

### Run the Test via CLI

```bash
curl -s "https://http-observatory.security.mozilla.org/api/v1/analyze?host=yourdomain.com" \
  -d "hidden=true&rescan=true" \
  -X POST | python3 -m json.tool | grep -E '"grade"|"score"'
```

---

## 9. Kali Linux — Manual Testing

### SQLMap — SQL Injection Testing

> Only test against your own application. SQLMap can cause significant
> database load and log noise.

```bash
# Test the WordPress login form for SQL injection
sqlmap -u "https://yourdomain.com/wp-login.php" \
       --data="log=admin&pwd=test&wp-submit=Log+In&redirect_to=%2Fwp-admin%2F&testcookie=1" \
       --cookie="wordpress_test_cookie=WP+Cookie+check" \
       --level=2 \
       --risk=1 \
       --batch \
       -o \
       --output-dir=testing/kali/sqlmap/
```

Expected result after hardening: `all tested parameters do not appear to be injectable`

### Metasploit — WordPress Brute Force Test

```bash
msfconsole -q

# WordPress login brute force module
use auxiliary/scanner/http/wordpress_login_enum
set RHOSTS yourdomain.com
set RPORT 443
set SSL true
set TARGETURI /YOUR_CUSTOM_LOGIN_PATH
set USER_FILE /usr/share/wordlists/metasploit/http_default_users.txt
set PASS_FILE /usr/share/wordlists/metasploit/http_default_pass.txt
set VERBOSE false
run
```

Expected result after hardening:
- Module cannot find the login page at `/wp-admin` or `/wp-login.php`
- If pointed at the custom login URL, Wordfence rate limiting and lockout
  prevent any successful brute force within the attempt window

### Hydra — SSH Brute Force Test

```bash
hydra -l ec2-user \
      -P /usr/share/wordlists/rockyou.txt \
      -s 2222 \
      -t 4 \
      YOUR_ELASTIC_IP ssh
```

Expected result after hardening:
- All password attempts fail because password authentication is disabled
- fail2ban bans the Kali IP after 3 failed attempts
- Hydra reports `[ERROR] could not connect to ssh://YOUR_ELASTIC_IP:2222`

### Dirb / Gobuster — Directory Enumeration

```bash
# Enumerate directories and files
gobuster dir \
  -u https://yourdomain.com \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,html,bak \
  -t 20 \
  -o testing/kali/gobuster_after.txt
```

Expected results after hardening:
- `/wp-admin` → 404
- `/wp-login.php` → 404
- `/xmlrpc.php` → 403
- `/readme.html` → 403
- `/wp-config.php` → 403
- No backup files (`.bak`, `.sql`) accessible

---

## 10. Pre and Post Hardening Comparison

| Test | Finding | Before hardening | After hardening |
|------|---------|-----------------|-----------------|
| Nmap | Ports exposed | 22, 80, 443, 3306 + possible others | 80, 443, 2222 only |
| Nmap | SSH version banner | Full version string visible | Suppressed |
| Nmap | MySQL externally reachable | Yes — port 3306 open | No — filtered |
| Nessus | Critical CVEs | Present | 0 |
| Nessus | High CVEs | Present | 0 |
| Nessus | TLS issues | TLS 1.0/1.1 enabled, weak ciphers | Resolved — A+ |
| WPScan | WordPress version | Exposed in page source | Removed |
| WPScan | Username enumeration | `admin` discoverable | Blocked |
| WPScan | XML-RPC | Enabled | Disabled — 403 |
| WPScan | Vulnerable plugins | Possibly present | 0 — auto-updates |
| WPScan | `/wp-admin/` | Accessible | 404 |
| Nikto | Server version | `Apache/2.4.x Amazon Linux` | `Apache` only |
| Nikto | Security headers | None present | All 6 headers present |
| Nikto | TRACE method | Enabled | Disabled |
| Wireshark | Login credentials visible | Yes over HTTP | No — TLS only |
| SSL Labs | TLS grade | C or below | A+ |
| Observatory | Header score | F | A+ |
| SQLMap | SQL injection | Not injectable (WordPress PDO) | Not injectable |
| Hydra | SSH brute force | Password auth available | Disabled — key only |
| Gobuster | Admin panel discoverable | Yes at `/wp-admin` | No — 404 |

---

## 11. Verification Checklist

| Tool | Test | Expected result |
|------|------|----------------|
| Nmap | `nmap -p- YOUR_ELASTIC_IP` | Only 80, 443, 2222 open |
| Nmap | `nmap -p 3306 YOUR_ELASTIC_IP` | filtered or closed |
| Nessus | Full scan after hardening | 0 Critical, 0 High |
| WPScan | `--enumerate u` | No users returned |
| WPScan | `--enumerate vp` | 0 vulnerable plugins |
| Nikto | Full scan | No critical findings |
| curl | `curl -I https://yourdomain.com/xmlrpc.php` | 403 Forbidden |
| curl | `curl -I https://yourdomain.com/wp-admin/` | 404 Not Found |
| curl | `curl -I http://yourdomain.com` | 301 → https:// |
| SSL Labs | `ssllabs.com` scan | A+ |
| Observatory | `observatory.mozilla.org` | A+ |
| Wireshark | Port 443 capture | TLS Application Data only |
| Hydra | SSH password brute force | All attempts fail immediately |

---

## References

- [Nmap Documentation](https://nmap.org/book/man.html)
- [Nessus Essentials](https://www.tenable.com/products/nessus/nessus-essentials)
- [WPScan Documentation](https://github.com/wpscanteam/wpscan)
- [Nikto](https://github.com/sullo/nikto)
- [Wireshark User Guide](https://www.wireshark.org/docs/wsug_html/)
- [SSL Labs Test](https://www.ssllabs.com/ssltest/)
- [Mozilla Observatory](https://observatory.mozilla.org)
- [Kali Linux Tools](https://www.kali.org/tools/)
- [Metasploit Documentation](https://docs.metasploit.com/)
- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)

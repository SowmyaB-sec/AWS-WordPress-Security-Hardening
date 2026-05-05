# 🔦 LAMP Stack — Installation & Hardening

This document covers the complete installation and security hardening of **Apache, MySQL, and PHP** on **Amazon Linux 2023**. All commands are tested for AL2023 using `dnf`.

The stack is installed in this order (Apache → MySQL → PHP) to make troubleshooting easier at each layer.

---

## Contents

| Section | Description |
|---------|-----------|
| [1. Install Apache](#1-install-apache) | Web server setup |
| [2. Apache Hardening](#2-apache-hardening) | Version hiding, security headers |
| [3. mod_security WAF](#3-mod_security-waf) | Web Application Firewall |
| [4. SSL/TLS with Let's Encrypt](#4-ssltls-with-lets-encrypt) | HTTPS and certificate management |
| [5. Apache Virtual Host](#5-apache-virtual-host-configuration) | WordPress-specific configuration |
| [6. Install MySQL](#6-install-mysql) | Database installation |
| [7. MySQL Hardening](#7-mysql-hardening) | Secure configuration |
| [8. Install PHP](#8-install-php) | PHP and extensions |
| [9. PHP Hardening](#9-php-hardening) | Secure `php.ini` settings |
| [10. Verify the Full Stack](#10-verify-the-full-stack) | End-to-end validation |
| [11. Verification Checklist](#11-verification-checklist) | Final confirmation |

---

## 1. Install Apache

```bash
sudo dnf install httpd -y
sudo systemctl enable --now httpd
sudo systemctl status httpd
```
### Confirm Apache is Responding

```bash
# From the instance itself
curl -I http://localhost

# From your local machine — should return the Apache test page
curl -I http://YOUR_ELASTIC_IP
```
### Set Correct Ownership of the Web Root

```bash
sudo chown -R ec2-user:apache /var/www/html
sudo chmod 2775 /var/www/html
find /var/www/html -type d -exec sudo chmod 2775 {} \;
find /var/www/html -type f -exec sudo chmod 0664 {} \;
```

## 2. Apache Hardening

All Apache configuration on Amazon Linux 2023 lives under `/etc/httpd/`. The main
configuration file is `/etc/httpd/conf/httpd.conf`. Additional configuration files
are loaded from `/etc/httpd/conf.d/`.

Create a dedicated security configuration file(so that hardening settings are kept separate and are easy to audit)

```bash
sudo nano /etc/httpd/conf.d/security.conf
```

```apache
# ── Server information hiding ─────────────────────────────────────────
# Hide Apache version number and OS from all HTTP response headers
ServerTokens Prod
ServerSignature Off

# Prevent PHP version disclosure via X-Powered-By header
# (Also set in php.ini - belt and braces)
Header always unset X-Powered-By

# ── Directory security ───────────────────────────────────────────────────────
# Disable directory listing globally — no browsing of directories
# without an index file
<Directory "/var/www/html">
    Options -Indexes -Includes
    AllowOverride All
    Require all granted
</Directory>

# ── HTTP security headers ───────────────────────────────────────────────────
# Force HTTPS for 1 year, include subdomains, allow browser preload
Header always set Strict-Transport-Security \
    "max-age=31536000; includeSubDomains; preload"

# Prevent the page being loaded in an iframe on another origin (clickjacking)
Header always set X-Frame-Options "SAMEORIGIN"

# Prevent MIME type sniffing
Header always set X-Content-Type-Options "nosniff"

# Enable browser XSS filter and block the page on detection
Header always set X-XSS-Protection "1; mode=block"

# Control how much referrer information is sent with requests
Header always set Referrer-Policy "strict-origin-when-cross-origin"

# Restrict access to browser features — camera, microphone, geolocation
Header always set Permissions-Policy \
    "geolocation=(), microphone=(), camera=(), payment=(), usb=()"


# Content Security Policy — restrict resource loading to own origin by default
# Adjust script-src and style-src if you use external CDNs
Header always set Content-Security-Policy \
    "default-src 'self'; \
     script-src 'self' 'unsafe-inline'; \
     style-src 'self' 'unsafe-inline'; \
     img-src 'self' data: https:; \
     font-src 'self'; \
     connect-src 'self'; \
     frame-ancestors 'self';"

# ── Request method restrictions/ Restrict dangerous HTTP methods ─────────────────────────────────────────────
# Only allow GET, POST, HEAD — disable TRACE, PUT, DELETE, OPTIONS
<LimitExcept GET POST HEAD>
    Require all denied
</LimitExcept>

# ── Protect Sensitive files ────────────────────────────────────────────────
# Block access to backup files, config files, logs, and other
# sensitive file types at the web server level
<FilesMatch "(^#.*#|\.(bak|conf|dist|fla|inc|ini|log|psd|sh|sql|sw[op])|~)$">
    Require all denied
</FilesMatch>

# Block access to .htaccess and .htpasswd files themselves
<Files ".ht*">
    Require all denied
</Files>

# ── Timeout settings/ Connection hardening ─────────────────────────────────────────────────────────
# Limit slow loris and connection-hold attacks
Timeout 60
KeepAliveTimeout 5
MaxKeepAliveRequests 100
```

```bash
sudo apachectl configtest && sudo systemctl reload httpd
```
---

## 3. mod_security WAF
Install and enable the OQASP Core Rule Set for web application Protection.
```bash
sudo dnf install mod_security mod_security_crs -y
```
Enable the module and CRS in a new config file:
```bash
sudo nano /etc/httpd/conf.d/modsecurity.conf
```


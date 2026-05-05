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
```apache
<IfModule security2_module>
    SecRuleEngine On
    SecRequestBodyAccess On
    SecRuleEngineDetectionMode DetectionOnly   # Change to On (Blocking) after testing

    # OWASP CRS
    IncludeOptional /etc/httpd/modsecurity.d/owasp-crs/*.conf
    IncludeOptional /etc/httpd/modsecurity.d/activated_rules/*.conf
</IfModule>
```
```bash
sudo apachectl configtest && sudo systemctl reload httpd
```

---

## 4. SSL/TLS with Let's Encrypt

### Prerequisites

Before running Certbot:
- Your domain's A record must point to the instance Elastic IP
- Port 80 must be open (Certbot uses HTTP-01 challenge)
- Apache must be running and responding on port 80

### Install Certbot

```bash
sudo dnf install augeas-libs -y
sudo python3 -m venv /opt/certbot/
sudo /opt/certbot/bin/pip install --upgrade pip
sudo /opt/certbot/bin/pip install certbot certbot-apache
sudo ln -s /opt/certbot/bin/certbot /usr/bin/certbot
```

### Obtain and Install Certificate

```bash
sudo certbot --apache -d yourdomain.com -d www.yourdomain.com
```

Certbot will:
1. Verify domain ownership via HTTP-01 challenge
2. Obtain a certificate from Let's Encrypt
3. Automatically modify your Apache VirtualHost to enable HTTPS
4. Configure an HTTP → HTTPS redirect

### Test Auto-Renewal

Let's Encrypt certificates expire every 90 days. Certbot installs a renewal timer
automatically. Test it:

```bash
sudo certbot renew --dry-run
```

### Verify the Renewal Timer is Active

```bash
sudo systemctl status certbot.timer
# or if using cron:
sudo crontab -l | grep certbot
```

### Harden the TLS Configuration

After Certbot runs, further tighten the TLS settings in
`/etc/httpd/conf.d/ssl.conf` or the VirtualHost file Certbot created:

```apache
# Disable old, insecure protocols — TLS 1.0 and 1.1 are deprecated
SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1

# Allow only strong cipher suites — forward secrecy first
SSLCipherSuite ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:\
ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:\
ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:\
DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384

# Server chooses cipher order (not client)
SSLHonorCipherOrder on

# Enable OCSP stapling — improves certificate validation performance
SSLUseStapling on
SSLStaplingCache "shmcb:/var/run/ocsp(128000)"

# Disable SSL compression — CRIME attack mitigation
SSLCompression off

# Enable HTTP/2 support
Protocols h2 http/1.1
```

```bash
sudo apachectl configtest && sudo systemctl reload httpd
```
---

## 5. Apache Virtual Host Configuration
Replace the default Apache test page with a clean VirtualHost configuration
for the WordPress site.
```bash
sudo nano /etc/httpd/conf.d/wordpress.conf
```
```apache
# HTTP → HTTPS Redirect
<VirtualHost *:80>
    ServerName yourdomain.com
    ServerAlias www.yourdomain.com
    Redirect permanent / https://yourdomain.com/
</VirtualHost>

# HTTPS VirtualHost
<VirtualHost *:443>
    ServerName yourdomain.com
    ServerAlias www.yourdomain.com
    DocumentRoot /var/www/html/wordpress

    SSLEngine on
    SSLCertificateFile    /etc/letsencrypt/live/yourdomain.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/yourdomain.com/privkey.pem

    <Directory /var/www/html/wordpress>
        Options -Indexes -Includes
        AllowOverride All
        Require all granted
    </Directory>

    # Prevent PHP execution in uploads directory
    <Directory /var/www/html/wordpress/wp-content/uploads>
        <Files "*.php">
            Require all denied
        </Files>
    </Directory>

    # Block author enumeration
    RewriteEngine On
    RewriteCond %{QUERY_STRING} ^author=\d
    RewriteRule ^ - [F,L]

    ErrorLog  /etc/httpd/logs/wordpress_error.log
    CustomLog /etc/httpd/logs/wordpress_access.log combined
</VirtualHost>
```
```bash
sudo apachectl configtest && sudo systemctl reload httpd
```
> **Note**: Create the ```/var/www/html/wordpress``` directory before proceding to the WordPress installation.

---

## 6. Install MySQL

### Install

```bash
sudo dnf install mysql-server -y
sudo systemctl enable mysqld
sudo systemctl start mysqld
sudo systemctl status mysqld
```

### Run the Secure Installation Script

This is a mandatory step. It removes the insecure defaults that MySQL ships with.

```bash
sudo mysql_secure_installation
```

Answer the prompts as follows:

| Prompt | Recommended answer |
|--------|-------------------|
| Set up VALIDATE PASSWORD component? | Yes |
| Password validation policy level | 2 (STRONG) |
| Remove anonymous users? | Yes |
| Disallow root login remotely? | Yes |
| Remove test database and access to it? | Yes |
| Reload privilege tables now? | Yes |

### Create the WordPress Database and User

```bash
sudo mysql -u root -p
```

At the MySQL prompt:

```sql
-- Create the WordPress database
CREATE DATABASE wordpress_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Create a dedicated WordPress user
-- Replace 'STRONG_PASSWORD_HERE' with a long, randomly generated password
-- Do not reuse any other password
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'STRONG_PASSWORD_HERE';

-- Grant only the privileges WordPress needs — nothing more
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER,
      CREATE TEMPORARY TABLES, LOCK TABLES
ON wordpress_db.* TO 'wp_user'@'localhost';

-- Apply the changes
FLUSH PRIVILEGES;

-- Confirm the user exists and has no global privileges
SHOW GRANTS FOR 'wp_user'@'localhost';

EXIT;
```

> The WordPress user has no `FILE`, `SUPER`, `PROCESS`, or global privileges.
> These are not needed for normal WordPress operation and removing them limits
> the damage possible if WordPress itself is compromised.
> ```wp_user``` has least privilege.

---
## 7. MySQL Hardening

After `mysql_secure_installation`, apply the following configuration hardening.

```bash
sudo nano /etc/my.cnf.d/security.cnf
```

Create this file with the following content:

```ini
[mysqld]

# ── Network hardening ─────────────────────────────────────────────────────
# Bind MySQL to localhost only — it must never listen on a public interface
bind-address = 127.0.0.1

# Disable loading local files from the client — prevents LOCAL INFILE attacks
local-infile = 0

# ── Connection limits ─────────────────────────────────────────────────────
# Limit simultaneous connections to prevent resource exhaustion
max_connections = 100

# Maximum failed connection attempts before temporary lockout
max_connect_errors = 10

# ── Query and logging hardening ────────────────────────────────────────────
# Log slow queries for performance and anomaly review
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2

# General query log — enable temporarily for auditing, disable in production
# general_log = 0
# general_log_file = /var/log/mysql/general.log

# ── Data at rest ──────────────────────────────────────────────────────────
# Enable binary logging for point-in-time recovery
log_bin = /var/log/mysql/mysql-bin.log
expire_logs_days = 7

# ── Security plugin ───────────────────────────────────────────────────────
# Validate password strength on all accounts
validate_password.policy = STRONG
validate_password.length = 12
validate_password.mixed_case_count = 1
validate_password.number_count = 1
validate_password.special_char_count = 1
```

```bash
# Create the MySQL log directory
sudo mkdir -p /var/log/mysql
sudo chown mysql:mysql /var/log/mysql

# Restart MySQL to apply changes
sudo systemctl restart mysqld

# Verify MySQL is only listening on localhost
sudo ss -tlnp | grep 3306
```

Expected output — address must be `127.0.0.1:3306`, not `0.0.0.0:3306`:

LISTEN  0  151  127.0.0.1:3306  0.0.0.0:*  users:(("mysqld",pid=XXXX,fd=XX))


### Verify No Anonymous Users Remain

```bash
sudo mysql -u root -p -e "SELECT User, Host FROM mysql.user;"
```

Expected: no rows with an empty `User` field. Only `root@localhost` and
`wp_user@localhost` should appear.

---

## 8. Install PHP
WordPress requires PHP and a specific set of extensions. Install them all in
one command to avoid dependency issues later.

```bash
sudo dnf install php php-mysqlnd php-fpm php-json php-curl php-gd \
     php-mbstring php-xml php-xmlrpc php-soap php-intl php-zip \
     php-opcache php-imagick -y
```

### Verify the Installation

```bash
php -v
```

Expected output:
PHP 8.x.x (cli) (built: ...) ...
### Confirm PHP is Integrated with Apache

```bash
# Create a test file
echo "<?php phpinfo(); ?>" | sudo tee /var/www/html/info.php

# Restart Apache
sudo systemctl restart httpd

# Test from local machine
curl http://YOUR_ELASTIC_IP/info.php | grep "PHP Version"
```

> **Remove this file immediately after testing** — `phpinfo()` exposes system
> configuration details that are valuable to an attacker.

```bash
sudo rm /var/www/html/info.php
```

---

## 8. PHP Hardening
The default `php.ini` on Amazon Linux 2023 is located at `/etc/php.ini`.
The following settings reduce information disclosure, prevent code injection
attacks, and enforce secure session handling.

```bash
sudo nano /etc/php.ini
```

Find and set each of the following values. Some will already exist and need
changing; others may need to be added.

### Information Disclosure

```ini
; Hide the PHP version from HTTP response headers and HTML output
expose_php = Off

; Do not display errors to the browser — log them instead
display_errors = Off
display_startup_errors = Off

; Log errors to file
log_errors = On
error_log = /var/log/php/php_errors.log

; Set the error reporting level (log everything, display nothing)
error_reporting = E_ALL
```

### Remote File Inclusion Prevention

```ini
; Prevent PHP from opening remote URLs with file functions
; Disables Remote File Inclusion (RFI) attacks
allow_url_fopen = Off
allow_url_include = Off
```

### Dangerous Function Restrictions

```ini
; Disable functions that allow arbitrary OS command execution
; These are never needed by WordPress or its plugins under normal operation
disable_functions = exec,passthru,shell_exec,system,proc_open,popen,\
                    curl_exec,curl_multi_exec,parse_ini_file,show_source,\
                    phpinfo,pcntl_exec,popen,proc_get_status
```

### File Upload Limits

```ini
; Allow uploads (WordPress requires this for media)
file_uploads = On

; Restrict upload size — adjust to match your WordPress media needs
upload_max_filesize = 10M
post_max_size = 12M

; Maximum number of files per upload
max_file_uploads = 10
```

### Execution Limits

```ini
; Maximum time a script is allowed to run
max_execution_time = 30

; Maximum time spent parsing input data
max_input_time = 30

; Maximum memory per script
memory_limit = 256M
```

### Session Security

```ini
; Prevent JavaScript from accessing session cookies (XSS mitigation)
session.cookie_httponly = 1

; Only send session cookies over HTTPS
session.cookie_secure = 1

; Prevent session fixation attacks
session.use_strict_mode = 1

; Regenerate session ID on every request
session.use_only_cookies = 1

; Restrict session cookie to the same site
session.cookie_samesite = Strict

; Use a strong hash for session IDs
session.sid_length = 48
session.sid_bits_per_character = 6
```

### OPcache Configuration

OPcache improves PHP performance and should be configured securely.

```ini
; Enable OPcache
opcache.enable = 1
opcache.enable_cli = 0

; Memory allocated for cached scripts
opcache.memory_consumption = 128

; Maximum number of files to cache
opcache.max_accelerated_files = 10000

; Revalidation interval in seconds — set to 0 in development, 60+ in production
opcache.revalidate_freq = 60

; Do not allow inclusion of PHP files not in the cache
opcache.enable_file_override = 0
```

### Create the PHP Log Directory

```bash
sudo mkdir -p /var/log/php
sudo chown apache:apache /var/log/php
sudo chmod 750 /var/log/php
```

### Restart Apache to Apply PHP Changes

```bash
sudo systemctl restart httpd
```

---

## 10. Verify the Full Stack
Run through these checks to confirm the entire LAMP stack is correctly installed
and hardened before proceeding to WordPress installation.

### Apache

```bash
# Confirm Apache is running
sudo systemctl is-active httpd

# Confirm HTTPS is working and certificate is valid
curl -I https://yourdomain.com

# Confirm HTTP redirects to HTTPS
curl -I http://yourdomain.com
# Expected: 301 Moved Permanently → Location: https://yourdomain.com/

# Confirm security headers are present
curl -I https://yourdomain.com | grep -E \
  "Strict-Transport|X-Frame|X-Content|X-XSS|Referrer|Permissions|Content-Security"

# Confirm Apache version is NOT disclosed
curl -I https://yourdomain.com | grep Server
# Expected: Server: Apache  (no version number)
```

### MySQL

```bash
# Confirm MySQL is running and listening on localhost only
sudo systemctl is-active mysqld
sudo ss -tlnp | grep 3306

# Confirm WordPress user can connect and access the database
mysql -u wp_user -p -e "SHOW DATABASES;" 2>/dev/null | grep wordpress_db

# Confirm anonymous users do not exist
sudo mysql -u root -p -e \
  "SELECT User, Host FROM mysql.user WHERE User='';"
# Expected: Empty set
```

### PHP

```bash
# Confirm PHP version
php -v

# Confirm expose_php is Off
php -r "echo ini_get('expose_php');"
# Expected: (empty — means Off)

# Confirm allow_url_include is Off
php -r "echo ini_get('allow_url_include');"
# Expected: (empty — means Off)

# Confirm dangerous functions are disabled
php -r "var_dump(function_exists('system'));"
# Expected: bool(false)
```

### SSL/TLS Grade

Visit the following URL and confirm an **A+ rating** is returned:
Common reasons for less than A+:
- TLS 1.0 or 1.1 still enabled — disable in `ssl.conf`
- HSTS header missing or `max-age` too short — minimum 31536000
- Weak cipher suites still enabled
- HSTS preload not configured

---

## 10. Verification Checklist

| Check | Command / Location | Expected result |
|-------|--------------------|----------------|
| Apache running | `systemctl is-active httpd` | `active` |
| HTTP → HTTPS redirect | `curl -I http://yourdomain.com` | `301` to `https://` |
| Server header clean | `curl -I https://yourdomain.com \| grep Server` | `Server: Apache` only |
| Directory listing off | Visit `https://yourdomain.com/wp-content/` | 403 Forbidden |
| Security headers present | `curl -I https://yourdomain.com` | All 6 headers visible |
| SSL Labs grade | ssllabs.com | A+ |
| MySQL on localhost only | `ss -tlnp \| grep 3306` | `127.0.0.1:3306` only |
| No anonymous MySQL users | `SELECT User FROM mysql.user WHERE User=''` | Empty set |
| WordPress DB user created | `SHOW GRANTS FOR 'wp_user'@'localhost'` | Limited grants only |
| PHP version hidden | `curl -I https://yourdomain.com \| grep PHP` | No result |
| PHP dangerous functions off | `php -r "var_dump(function_exists('system'));"` | `bool(false)` |
| PHP error display off | `php -r "echo ini_get('display_errors');"` | Empty |

---

## References

- [Apache HTTP Server Documentation](https://httpd.apache.org/docs/2.4/)
- [Apache Security Tips — Official](https://httpd.apache.org/docs/2.4/misc/security_tips.html)
- [MySQL 8.0 Security Guide](https://dev.mysql.com/doc/refman/8.0/en/security.html)
- [PHP Security Manual](https://www.php.net/manual/en/security.php)
- [CIS Apache HTTP Server Benchmark](https://www.cisecurity.org/benchmark/apache_http_server)
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
- [Let's Encrypt — Certbot Instructions](https://certbot.eff.org/instructions?os=centosrhel8)
- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/)

Common reasons for less than A+:
- TLS 1.0 or 1.1 still enabled — disable in `ssl.conf`
- HSTS header missing or `max-age` too short — minimum 31536000
- Weak cipher suites still enabled
- HSTS preload not configured


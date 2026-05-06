# 🌐 WordPress — Deployment & Hardening

This document covers the deployment of WordPress on the hardened LAMP stack and applies comprehensive **application-level security** controls.

---

## Contents

| Section | Description |
|---------|-----------|
| [1. Download and Install WordPress](#1-download-and-install-wordpress) | Core installation |
| [2. Configure wp-config.php](#2-configure-wp-configphp) | Database + security keys |
| [3. Harden wp-config.php](#3-harden-wp-configphp) | Security constants |
| [4. File Permission Hardening](#4-file-permission-hardening) | Secure permissions |
| [5. .htaccess Hardening](#5-htaccess-hardening) | Web server protections |
| [6. Admin URL Relocation](#6-wordpress-admin-url-relocation) | Hide login page |
| [7. Wordfence Security](#7-wordfence-security) | WAF and malware protection |
| [8. MFA with WP 2FA](#8-mfa-with-wp-2fa) | Two-factor authentication |
| [9. BackWPup — Encrypted Backups to S3](#9-backwpup--backup-to-aws-s3) | Automated backups |
| [10. Additional Hardening](#10-additional-hardening) | REST API, version hiding, etc. |

---

## 1. Download and Install WordPress

```bash
cd /var/www/html

# Download and extract
sudo wget https://wordpress.org/latest.tar.gz
sudo tar -xzf latest.tar.gz

# Move files into place (recommended structure)
sudo rsync -av wordpress/ ./
sudo rm -rf wordpress latest.tar.gz

# Set correct ownership
sudo chown -R apache:apache /var/www/html
sudo chmod -R 755 /var/www/html
```
---

## 2. Configure wp-config.php
```bash
sudo cp wp-config-sample.php wp-config.php
sudo nano wp-config.php
```
Update the database settings:
```PHP
define( 'DB_NAME', 'wordpress_db' );
define( 'DB_USER', 'wp_user' );
define( 'DB_PASSWORD', 'STRONG_PASSWORD_HERE' );
define( 'DB_HOST', 'localhost' );
define( 'DB_CHARSET', 'utf8mb4' );
```
Genereate fresh security keys:
```bash
curl -s https://api.wordpress.org/secret-key/1.1/salt/
```
Replace the entire keys block with the output.

---

## 3. Harden wp-config.php
Add these constants above the line ```/* That's all, stop editing! */```

```PHP
// Security Hardening
$table_prefix = 'xk7_';                    // Non-default prefix

define( 'DISALLOW_FILE_EDIT', true );      // Disable theme/plugin editor
define( 'DISALLOW_FILE_MODS', true );      // Disable plugin/theme install & updates via admin
define( 'FORCE_SSL_ADMIN', true );         // Force HTTPS in admin
define( 'WP_DEBUG', false );
define( 'WP_DEBUG_LOG', false );
define( 'WP_DEBUG_DISPLAY', false );

define( 'WP_AUTO_UPDATE_CORE', true );     // Minor/core security updates
define( 'WP_POST_REVISIONS', 5 );
define( 'EMPTY_TRASH_DAYS', 14 );

define( 'DISABLE_WP_CRON', true );         // Use system cron instead
define( 'WP_MEMORY_LIMIT', '256M' );
```
System Cron Setup (recommended):
```bash
sudo crontab -u apache -e
```
Add:
```cron
*/5 * * * * /usr/bin/php /var/www/html/wp-cron.php >/dev/null 2>&1
```

---

## 4. File Permission Hardening
```bash
sudo chown -R apache:apache /var/www/html

# Directories
sudo find /var/www/html -type d -exec chmod 755 {} \;

# Files
sudo find /var/www/html -type f -exec chmod 644 {} \;

# wp-config.php - strictest permissions
sudo chmod 600 /var/www/html/wp-config.php

# Uploads directory (writable but controlled)
sudo chmod 755 /var/www/html/wp-content/uploads
```

---

## 5. .htaccess Hardening
```bash
sudo nano /var/www/html/.htaccess
```
Recommended hardened .htaccess
```# WordPress Core Rewrites
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteBase /
RewriteRule ^index\.php$ - [L]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . /index.php [L]
</IfModule>

# Block XML-RPC (high abuse target)
<Files xmlrpc.php>
    Require all denied
</Files>

# Protect sensitive files
<Files wp-config.php>
    Require all denied
</Files>

<Files .htaccess>
    Require all denied
</Files>

# Block dangerous file types
<FilesMatch "\.(log|sql|bak|backup|conf|env|git|svn)$">
    Require all denied
</FilesMatch>

# Prevent PHP execution in uploads
<Directory wp-content/uploads>
    <Files "*.php">
        Require all denied
    </Files>
</Directory>

# Block author enumeration
RewriteCond %{QUERY_STRING} author=\d
RewriteRule ^ - [F,L]

# Block readme/license files (version disclosure)
<FilesMatch "^(readme|license|wp-activate)\.(html|txt|php)$">
    Require all denied
</FilesMatch>

# Security headers (backup layer)
<IfModule mod_headers.c>
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-Content-Type-Options "nosniff"
</IfModule>

Options -Indexes
```
---

## 6. WordPress Admin URL Relocation
Use the **WPS Hide Login** plugin to change the login URL from ```/wp-admin``` and ```/wp-login.php```.

**Recommended new login path**: Something non-obvious like ```/secure-login``` or ```/portal-2025```.

---

## 7. WordFence Security
Install **Wordfence Security** and configure
- **Firewall Mode**: Enabled and Protecting (Extended Protection)
- **Brute Force Protection**: Lock after 4–5 attempts
- **Rate Limiting**: Enable
- Run initial full scan and schedule daily scans

---

## 8. MFA with WP 2FA 
Install **WP 2FA** plugin and enforce for all Administrators (Grace period = 0 days).

Use **TOTP** (Authenticator apps) + backup codes.

---

## 9. BackWPup — Encrypted Backups to S3
Install **BackWPup** and configure a daily job:

- Backup: Database + Files
- Destination: Amazon S3 (use IAM Role — **no access keys**)
- Encryption: Enabled
- Retention: 14 days

---

## 10. Additional Hardening
- Disable application passwords (via Wordfence or custom code)
- Restrict REST API for unauthenticated users
- Remove WordPress version from headers and feeds
- Rename default admin username
- Keep only essential, actively maintained plugins


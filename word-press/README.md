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
| [11. Verification Checklist](#11-verification-checklist) | Final validation |

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



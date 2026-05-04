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


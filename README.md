# Secure LAMP + WordPress on AWS EC2
## Project Overview
> This project demonstrates **end-to-end cloud infrastructure security** on AWS. It showcases the secure, scalable deployment, configuration, and security hardening of WordPress application hosted on Amazon Web Services (AWS) using an Infrastructure-as-a-Service (IaaS) model. The project includes full-stack hardening across cloud infrastructure, operating system, and WordPress application layer, supported by monitoring, HTTPS enforcement, and security testing.
> 
![AWS](https://img.shields.io/badge/AWS-EC2-FF9900?style=flat&logo=amazon-aws)
![WordPress](https://img.shields.io/badge/WordPress-Hardened-21759B?style=flat&logo=wordpress)
![Apache](https://img.shields.io/badge/Apache-Secured-CA2136?style=flat&logo=apache)
![MySQL](https://img.shields.io/badge/MySQL-Hardened-4479A1?style=flat&logo=mysql)
![PHP](https://img.shields.io/badge/PHP-8.x-777BB4?style=flat&logo=php)
![License](https://img.shields.io/badge/License-MIT-green)

---

## Security Challenges & Threat Model
Deploying a LAMP + WordPress stack on a public cloud infrastructure introduces attack vectors.
- Brute-force attacks on SSH and admin panels.
- Web application exploits (SQLi, XSS, RCE via vulnerable plugins).
- Misconfigurations leading to data exposure.
- Supply-chain risks from plugins and themes.
- DDoS and volumetric attacks.
- Insufficient logginf and slow breach detection.

A **STRIDE** + OSWAP Top 10** (Web and Cloud) approach was used to identify risks and implement layered controls across Network, Compute, Application, Identity, Data, and Observability layers.

--- 

## Architecture Overview
- **Platform**: AWS EC2 (Amazon Linux 2023)
- **Stack**: Apache 2.4 + MySQL 8.0 + php 8.x + WordPress 6.x
- **Design Selection**:
    - EC2 chosen for full control and visibility into security configurations.
    - Single instance with local database
    - Public-facing application protected by mulitple security layers.

![Architecture Diagram]
## Quick Start
### Prerequisites
- An **AWS Account** with necessary Permissions (EC2, VPC, IAM, S3, CloudWatch)
- **AWS CLI** installed and configured locally.
- A valid **SSH key pair** created and registered in AWS for EC2 access.
- A domain name pointed to the EC2 instance's Elastic Ip (required for HTTPS/Let's Encrypt)
- A local machine with **SSH** and **Git** installed.
- Basic knowledge of Linux commands
- (Optional - but recommended) A **Datadog** account for extended monitoring and altering.

### Setup Steps
1. Clone this repository
2. Provision and harden the EC2 instance ('infrastructure/README.md')
3. Install and secure the LAMP stack ('lamp-stack/README.md')
4. Deploy and harden WordPress ('wordpress/README.md')
5. Set up monitoring and alterting ('monitoring/README.md')
6. Run security tests and validation ('testing/README.md')

> **Important:** Only perform security testing on infrastructure you own. Clean up resources after use to avoid unnecessary costs.

## Key Hardening Features
### Infrastructure Security (AWS)
- Hardened Amazon Linux 2023 instance with minimal footprint
- Restrictive Security Groups (80, 443, custom SSH port with IP allowlisting)
- SSH: Key-only authentication, root login disabled, Fail2Ban protection
- OS-level firewall (UFW) with default-deny policy
- Least-privilege IAM role attached to instance
- Automatic security patching enabled
- AWS Inspector for continuous vulnerability scanning

### LAMP Stack Hardening
- Apache: Server tokens hidden, directory listing disabled, strong HTTP security headers
- TLS: Let's Encrypt certificates with automatic HTTP-to-HTTPS redirect and HSTS
- mod_security WAF with OWASP Core Rule Set
- MySQL: Hardened configuration, dedicated least-privilege user, remote access disabled
- PHP: Version exposure disabled, dangerous functions restricted

### WordPress Application Security
- Hardening 'wp-config.php' (custom table prefix, debug disabled, SSL enforcement).
- Admin login URL changed + MFA/OTP.
- Wordfence Security (WAF, firewall, malware scanner) in blocking mode.
- XML-RPC idsabled, strict file permissions, automatic updates enabled.
- Encrypted daily backups to AWS S3.
  
### Monitoring & Alterting
- AWS CloudWatch for metrics, logs, and intelligent alarms.
- Datadog integration for advanced monitoring and anomaly detection
- Datadog integration for advanced monitoring and anomoly detection.


---

## Technologies and Tools Used
### Cloud & Infrastructure
| Tool | Purpose |
|--------------|--------------|
|AWS EC2 | Compute |

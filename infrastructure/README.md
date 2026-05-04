# Infrastructure - EC2 Provisioning & Hardening
This document details the provisioning and hardeing of the EC2 infrastructure and operating system layer (Amazon Linux 2023) before installing the LAMP Stack and WordPress

Emphasis is placed on **least privilege**, attack surface reduction, and defence-in-depth.
---

## 1. Launch EC2 Instance
### Recommended Settings
| Setting          | Value                                      |
|------------------|--------------------------------------------|
| AMI              | Amazon Linux 2023 (64-bit x86)             |
| Instance Type    | t3.micro (or t2.micro for Free Tier)       |
| Storage          | 20–30 GB gp3 EBS volume                    |
| Network          | Public subnet in a dedicated VPC           |

**Best Practice**: Use a dedicated VPC and public subnet for this project. Avoid using the default VPC in production-like setups.
### Launch via AWS Console or CLI

(Keep your existing detailed steps — they are good. Consider adding a note about enabling **IMDSv2** under Advanced Details for better security.)


## 2. Security Group Configuration

**Before Hardening** (Insecure baseline):
- SSH (22) open to `0.0.0.0/0`
- MySQL (3306) potentially exposed

**Hardened Inbound Rules**:

| Protocol | Port | Source          | Purpose |
|----------|------|-----------------|---------|
| TCP      | 443  | 0.0.0.0/0       | HTTPS |
| TCP      | 80   | 0.0.0.0/0       | HTTP (redirect to HTTPS) |
| TCP      | 2222 | YOUR_ADMIN_IP/32| SSH (custom port) |

**Outbound Rules**: Allow 80/443 (and 587 if using email). Consider restricting further in production.

> **Important**: No inbound rule for port 3306. MySQL must remain bound to `127.0.0.1`.

---

## 3. Elastic IP
Assign a static public IP so that your domain DNS record and SSL certificate remain valid if the instance is stopped and restarted.
'''
```bash
# Allocate an Elastic IP
aws ec2 allocate-address --domain vpc

# Associate it with your instance
aws ec2 associate-address \
  --instance-id i-XXXXXXXXXXXXXXXX \
  --allocation-id eipalloc-XXXXXXXX
```

Point your domain's A record to this Elastic IP before running Certbot for Let's Encrypt in the LAMP stack setup.

---

## 4. IAM Role
The EC2 instance needs an IAM role to communicate with CloudWatch and S3 without storing access keys on the instance. Never put AWS access keys directly on an EC2 instance.
### Create the IAM Role via AWS Console
1. Navigate to **IAM → Roles → Create role**
2. Trusted entity: **AWS service → EC2**
3. Attach the following managed policies:
   - `CloudWatchAgentServerPolicy` — allows the CloudWatch agent to publish metrics and logs
   - `AmazonSSMManagedInstanceCore` — enables Systems Manager Session Manager (alternative to direct SSH)
4. For S3 backup access, create a custom inline policy (see below)
5. Name the role: `EC2-CloudWatch-S3-Role`
6. Attach the role to your EC2 instance: **EC2 → Instance → Actions → Security → Modify IAM role**

### Custom S3 Backup Policy

This policy grants the instance write-only access to a specific S3 bucket for BackWPup backups — nothing broader.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::YOUR-BACKUP-BUCKET-NAME",
        "arn:aws:s3:::YOUR-BACKUP-BUCKET-NAME/*"
      ]
    }
  ]
}
```

Replace `YOUR-BACKUP-BUCKET-NAME` with your actual S3 bucket name.

**Recommended SSH alternative:** AmazonSSMManagedInstanceCore for Session manager

---

## 5. Connect to the Instance

```bash
# Set correct permissions on the key file
chmod 400 YOUR_KEY.pem

# Connect (use your Elastic IP and the default Amazon Linux 2023 user)
ssh -i YOUR_KEY.pem -p 22 ec2-user@YOUR_ELASTIC_IP
```

> After SSH hardening in Section 7, the port changes to 2222 and the user remains `ec2-user`.

---

## 6. OS Updates and Patching

Always update the system immediately after first login, before installing anything else.

### Manual Update

```bash
sudo dnf update -y
sudo dnf upgrade -y
```

### Enable Automatic Security Updates

Amazon Linux 2023 uses `dnf-automatic` for unattended updates.

```bash
# Install dnf-automatic
sudo dnf install dnf-automatic -y

# Edit the configuration
sudo nano /etc/dnf/automatic.conf
```

Set the following values in `automatic.conf`:

```ini
[commands]
upgrade_type = security
apply_updates = yes
emit_via = stdio

[email]
# Optional: configure email notifications
email_from = dnf-automatic@yourdomain.com
email_to = admin@yourdomain.com
email_host = localhost
```

```bash
# Enable and start the timer
sudo systemctl enable --now dnf-automatic.timer

# Verify it is active
sudo systemctl status dnf-automatic.timer
```

---

## 7. SSH Hardening

This is one of the highest-impact hardening steps. SSH is the primary remote access method and a constant target for brute-force attacks.

### Edit the SSH Daemon Configuration

```bash
sudo nano /etc/ssh/sshd_config
```

Apply the following changes:
Change default port — reduces automated scan noise significantly
Port 2222
Disable root login entirely
PermitRootLogin no
Disable password authentication — key-based only
PasswordAuthentication no ChallengeResponseAuthentication no
Ensure public key authentication is enabled
PubkeyAuthentication yes
Restrict login to specific user
AllowUsers ec2-user
Limit authentication attempts
MaxAuthTries 3
Disconnect idle sessions after 5 minutes
ClientAliveInterval 300 ClientAliveCountMax 2
Disable X11 forwarding — not needed
X11Forwarding no
Disable agent forwarding unless explicitly needed
AllowAgentForwarding no
Log level — INFO is sufficient, VERBOSE for troubleshooting
LogLevel INFO
Use only modern, strong key exchange algorithms
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,ecdh-sha2-nistp256 Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com MACs hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com

### Validate and Restart SSH

```bash
# Always validate the config before restarting — a broken sshd_config will lock you out
sudo sshd -t

# If no errors are reported, restart the service
sudo systemctl restart sshd

# Verify it is listening on the new port
sudo ss -tlnp | grep 2222
```

> **Critical:** Before closing your current SSH session, open a second terminal window and confirm you can connect on port 2222. If you cannot connect, you will need to use the AWS EC2 Instance Connect or Systems Manager Session Manager to recover access.

```bash
# Test connection from a new terminal before closing the old one
ssh -i YOUR_KEY.pem -p 2222 ec2-user@YOUR_ELASTIC_IP
```

**Remember** Changing the port reduces noise from automate scanners but is **not** security but obscurity - it must be combined with other controls.
**Test the new SSH configuration before closing the current session.**


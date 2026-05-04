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

1. Navigate to **EC2 → Launch Instance**
2. Name the instance: `lamp-wordpress-secure`
3. Select **Amazon Linux 2023 AMI**
4. Choose instance type (`t2.micro` for free tier)
5. Under **Key pair**, select an existing key pair or create a new one — download the `.pem` file and store it securely
6. Under **Network settings**, select your VPC and a public subnet
7. Configure Security Group (see Section 2)
8. Under **Advanced details**, attach your IAM role (see Section 4)
9. Click **Launch instance**

```bash
aws ec2 run-instances \
  --image-id ami-0c101f26f147fa7fd \
  --instance-type t2.micro \
  --key-name YOUR_KEY_PAIR_NAME \
  --security-group-ids sg-XXXXXXXX \
  --subnet-id subnet-XXXXXXXX \
  --iam-instance-profile Name=EC2-CloudWatch-S3-Role \
  --block-device-mappings '[{"DeviceName":"/dev/xvda","Ebs":{"VolumeSize":20,"VolumeType":"gp3"}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lamp-wordpress-secure}]'
```

> Replace `ami-0c101f26f147fa7fd` with the correct Amazon Linux 2023 AMI ID for your region. Find the latest at: **EC2 → AMI Catalog → Amazon Linux 2023**.

---

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
```
bash
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
- **Change default port — reduces automated scan noise significantly** - Port 2222
- **Disable root login entirely** - PermitRootLogin no
- **Disable password authentication - key-based only**
PasswordAuthentication no <br>
ChallengeResponseAuthentication no
- **Ensure public key authentication is enabled** 
PubkeyAuthentication yes
- **Restrict login to specific user**
AllowUsers ec2-user
- **Limit authentication attempts**
MaxAuthTries 3
- **Disconnect idle sessions after 5 minutes**
ClientAliveInterval 300 <br>
ClientAliveCountMax 2
- **Disable X11 forwarding — not needed**
X11Forwarding no
- **Disable agent forwarding unless explicitly needed**
AllowAgentForwarding no
- **Log level — INFO is sufficient, VERBOSE for troubleshooting**
LogLevel INFO
- **Use only modern, strong key exchange algorithms**
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

**Remember:** Changing the port reduces noise from automate scanners but is **not** security but obscurity - it must be combined with other controls.

**Test the new SSH configuration before closing the current session.**

## 8. Fail2Ban
fail2ban monitors log files and automatically bans IP addresses that show signs of brute-force behaviour.

### Install

```bash
sudo dnf install fail2ban -y
```
### Configure

```bash
# Copy the default config to a local override — never edit jail.conf directly

sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

sudo nano /etc/fail2ban/jail.local
```

Apply the following settings in 'jail.local':

```
ini
[DEFAULT]
# Ban duration — 1 hour
bantime  = 3600

# Time window to count failures
findtime = 600

# Number of failures before ban
maxretry = 5

# Backend — systemd for Amazon Linux 2023
backend = systemd

# Email notification (optional)
destemail = admin@yourdomain.com
sender    = fail2ban@yourdomain.com

[sshd]
enabled  = true
port     = 2222
logpath  = %(sshd_log)s
maxretry = 3
bantime  = 86400

[apache-auth]
enabled  = true
port     = http,https
logpath  = %(apache_error_log)s

[apache-badbots]
enabled  = true
port     = http,https
logpath  = %(apache_access_log)s
maxretry = 2
[apache-noscript]
enabled  = true

[apache-overflows]
enabled  = true
```
### Enable and Start

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# Check status
sudo fail2ban-client status

# Check the SSH jail specifically
sudo fail2ban-client status sshd

# View banned IPs
sudo fail2ban-client status sshd | grep "Banned IP"
```

---

## 9. UFW Firewall
UFW provides an OS-level firewall as a second layer of defence behind the AWS Security Group. Even if a Security Group rule is accidentally changed, UFW prevents unwanted traffic from reaching services.

```bash
# Install UFW
sudo dnf install ufw -y

# Set default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow the custom SSH port
sudo ufw allow 2222/tcp

# Allow web traffic
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Enable UFW
sudo ufw enable

# Verify rules
sudo ufw status verbose

Expected output after enabling 
Status: active
|To           |      Action  |    From   |
|-------------|--------------|-----------|
|2222/tcp     |    ALLOW IN  |  Anywhere |
|80/tcp       |    ALLOW IN  |  Anywhere |
|443/tcp      |    ALLOW IN  |  Anywhere |
```

**ALTERNATIVE METHOD**
**Use Host Firewall: firewalld (recommended for AL2023)**
> **NOTE**: On Amazon Linux 2023, AWS Security Groups provide a network boundary. A host-based firewall adds an extra layer of defense.

```bash
sudo dnf install firewalld -y
sudo systemctl enable --now firewalld
# Add rules
sudo firewall-cmd --permanent --add-port=2222/tcp
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload

sudo firewall-cmd --list-all
```

**Alternatively, if you prefer simplicity, you can primarily rely on Securoty Groups and skip host firewall for this project**


## 10. Disable Unused Services

Every running service is a potential attack surface. Audit what is running and disable anything not required.

```bash
# List all enabled services
sudo systemctl list-unit-files --state=enabled

# Common services to disable on a minimal web server
sudo systemctl disable --now bluetooth
sudo systemctl disable --now avahi-daemon
sudo systemctl disable --now cups
sudo systemctl disable --now postfix    # Unless you need local mail delivery
sudo systemctl disable --now rpcbind

# Confirm the services are stopped
sudo systemctl is-active bluetooth avahi-daemon cups

sudo systemctl list-unit-files --state=enabled
```

> DO NOT DISABLE **`sshd`, `crond`, `rsyslog`, `amazon-ssm-agent`, or `amazon-cloudwatch-agent`** - These are needed.

## 11. AWS CloudWatch Agent

The CloudWatch agent collects system metrics and log files that are not available from EC2 by defauld, including memory usage, disk usage, and applicatio logs.

### Install the Agent

```bash
sudo dnf install amazon-cloudwatch-agent -y
```

### Configure the Agent

Run the configuration wizard, which generated a JSON config file interactively:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

The following are to recommended to be said for the questions asked by the wizard for this project:
- Operating system: **Linux**
- Running on EC2: **Yes**
- StatsD daemon: **No**
- CollectD: **No**
- Host metrics: **Yes**
- CPU metrics per core: **No**
- Add EC2 dimensions: **Yes**
- Aggregation interval: **60 seconds**
- Memory metrics: **Yes**
- Disk metrics: **Yes**
- Disk paths: '/'
- Disk I/O metrics: **Yes**
- Network metrics: **Yes**
- Collect logs: **Yes**

When prompted to add log files, add:
| Log file path | Log group name | Log stream name |
|---------------|---------------|-----------------|
| '/var/log/messages' | '/ec2/system/messages' | '{instance_id}' |
| '/var/log/secure' | '/ec2/system/secure' | '{instance_id}' |
| '/etc/httpd/logs/access_log' | '/ec2/apache/access' | '{instance_id}' |
| '/etc/httpd/logs/error_log' | '/ec2/apache/error' | '{instance_id}' |

### Start the Agent

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
  -s

sudo systemctl enable amazon-cloudwatch-agent
sudo systemctl status amazon-cloudwatch-agent

### Create CloudWatch Alarms
Run these AWS CLI commands to create the core alarms. Replace 'YOUR_SNS_ARN' with an SNS topic ARN connected to your email.

```bash
# High CPU alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "EC2-HighCPU" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions YOUR_SNS_ARN \
  --dimensions Name=InstanceId,Value=i-XXXXXXXXXXXXXXXX

# High disk alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "EC2-HighDisk" \
  --metric-name disk_used_percent \
  --namespace CWAgent \
  --statistic Average \
  --period 300 \
  --threshold 85 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions YOUR_SNS_ARN \
  --dimensions Name=InstanceId,Value=i-XXXXXXXXXXXXXXXX Name=path,Value=/ Name=fstype,Value=xfs
  
```
### Create a Metric Filter for SSH Failures

```bash
# Create the metric filter against the /ec2/system/secure log group
aws logs put-metric-filter \
  --log-group-name "/ec2/system/secure" \
  --filter-name "SSHFailedLogins" \
  --filter-pattern "Failed password" \
  --metric-transformations \

metricName=SSHFailedLoginCount,metricNamespace=CustomMetrics,metricValue=1

# Create an alarm on that metric
aws cloudwatch put-metric-alarm \
  --alarm-name "SSH-BruteForce-Detected" \
  --metric-name SSHFailedLoginCount \
  --namespace CustomMetrics \
  --statistic Sum \
  --period 300 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions YOUR_SNS_ARN

  ```

  ---

  ## 12. AWS Inspector
AWS Inspector performs automated vulnerability assessments against the EC2 instance using the same CVE databases that professional scanners use.

### Enable via AWS Console

1. Navigate to **AWS Inspector → Get started**
2. Select **Enable Inspector**
3. Inspector will automatically discover your EC2 instances
4. Two assessment types run automatically:
   - **Network Reachability** — checks which ports are reachable from the internet
   - **Common Vulnerabilities and Exposures (CVE)** — scans installed packages for known CVEs
  
### Review Findings

1. Navigate to **Inspector → Findings**
2. Filter by severity: Critical, High, Medium
3. Remediate Critical and High findings before proceeding to application setup
4. Re-scan after remediation to confirm resolution

---

## 13. Verification Checklist

Run through this checklist after completing all sections above. Every item should be confirmed before moving on to LAMP stack installation.

```bash
# 1. SSH accessible on port 2222 with key only
ssh -i YOUR_KEY.pem -p 2222 ec2-user@YOUR_ELASTIC_IP

# 2. Root login denied
ssh -i YOUR_KEY.pem -p 2222 root@YOUR_ELASTIC_IP
# Expected: "Permission denied, please try again" or connection refused

#  3. Password authentication denied
ssh -p 2222 ec2-user@YOUR_ELASTIC_IP
# Expected: "Permission denied (publickey)"

# 4. UFW is active with correct rules
sudo ufw status verbose

# 5. fail2ban is running
sudo fail2ban-client status

# 6. Automatic updates timer active
sudo systemctl status dnf-automatic.timer

# 7. CloudWatch agent running
sudo systemctl status amazon-cloudwatch-agent

# 8. Port 3306 not reachable from outside
# Run from your local machine:
nmap -p 3306 YOUR_ELASTIC_IP
# Expected: filtered or closed

# 9. Only expected ports open
nmap -p- YOUR_ELASTIC_IP
# Expected: only 80, 443, 2222

# 10. Confirm IMDSv2 is required (hop limit = 2)

**Remember to Cleanup/terminate instance when done**
```

---
 ## References 
- [Amazon Linux 2023 Documentation](https://docs.aws.amazon.com/linux/al2023/ug/what-is-amazon-linux.html)
- [AWS EC2 Security Best Practices](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security.html)
- [AWS CloudWatch Agent Setup](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html)
- [CIS Amazon Linux 2 Benchmark](https://www.cisecurity.org/benchmark/amazon_linux) *(closest available to AL2023)*
- [fail2ban Documentation](https://www.fail2ban.org/wiki/index.php/MANUAL_0_8)
- [AWS Inspector Documentation](https://docs.aws.amazon.com/inspector/latest/user/getting_started_tutorial.html)




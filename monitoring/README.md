# 📊 Monitoring — CloudWatch & Datadog

This document covers the monitoring stack used to observe infrastructure health,
detect anomalies, and generate alerts. Two monitoring layers are used: AWS
CloudWatch for native infrastructure metrics and log aggregation, and Datadog
for deeper application-level visibility and anomaly detection.

CloudWatch agent installation was covered in `infrastructure/README.md` Section 11.
This document covers the dashboard configuration, log metric filters, alarms,
and Datadog integration.

---

## Contents

| Section | What it covers |
|---------|---------------|
| [1. CloudWatch Dashboards](#1-cloudwatch-dashboards) | Creating a monitoring dashboard |
| [2. Log Groups and Metric Filters](#2-log-groups-and-metric-filters) | Turning log events into metrics |
| [3. CloudWatch Alarms Summary](#3-cloudwatch-alarms-summary) | All alarms configured for this project |
| [4. SNS Email Notifications](#4-sns-email-notifications) | Wiring alarms to email alerts |
| [5. Datadog Agent Setup](#5-datadog-agent-setup) | Installing and configuring the Datadog agent |
| [6. Datadog Integrations](#6-datadog-integrations) | Apache, MySQL, and system integrations |
| [7. Datadog Monitors](#7-datadog-monitors) | Anomaly detection and threshold monitors |
| [8. Log Review Procedures](#8-log-review-procedures) | Manual log inspection commands |
| [9. Verification Checklist](#9-verification-checklist) | Confirm monitoring is fully active |

---

## 1. CloudWatch Dashboards

### Create a Dashboard via AWS Console

1. Navigate to **CloudWatch → Dashboards → Create dashboard**
2. Name: `LAMP-WordPress-Security`
3. Add the following widgets:

| Widget type | Metric | Purpose |
|-------------|--------|---------|
| Line graph | `AWS/EC2 → CPUUtilization` | Track CPU over time |
| Line graph | `CWAgent → mem_used_percent` | Memory usage |
| Line graph | `CWAgent → disk_used_percent` | Disk usage on `/` |
| Line graph | `AWS/EC2 → NetworkIn / NetworkOut` | Traffic volume |
| Number | `CustomMetrics → SSHFailedLoginCount` | Brute-force attempts |
| Number | `CWAgent → apache_requests_per_sec` | Request rate |
| Alarm status | All alarms | At-a-glance alarm state |

### Create the Dashboard via AWS CLI

```bash
aws cloudwatch put-dashboard \
  --dashboard-name "LAMP-WordPress-Security" \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "properties": {
          "title": "CPU Utilization",
          "metrics": [["AWS/EC2","CPUUtilization",
            "InstanceId","i-XXXXXXXXXXXXXXXX"]],
          "period": 300,
          "stat": "Average",
          "view": "timeSeries"
        }
      },
      {
        "type": "metric",
        "properties": {
          "title": "Memory Used %",
          "metrics": [["CWAgent","mem_used_percent",
            "InstanceId","i-XXXXXXXXXXXXXXXX"]],
          "period": 300,
          "stat": "Average",
          "view": "timeSeries"
        }
      },
      {
        "type": "metric",
        "properties": {
          "title": "Disk Used %",
          "metrics": [["CWAgent","disk_used_percent",
            "InstanceId","i-XXXXXXXXXXXXXXXX",
            "path","/","fstype","xfs"]],
          "period": 300,
          "stat": "Average",
          "view": "timeSeries"
        }
      }
    ]
  }'
```

---

## 2. Log Groups and Metric Filters

The CloudWatch agent ships the following logs to CloudWatch Log Groups
(configured in `infrastructure/README.md`):

| Log Group | Source file | What it captures |
|-----------|------------|-----------------|
| `/ec2/system/messages` | `/var/log/messages` | General system events |
| `/ec2/system/secure` | `/var/log/secure` | Authentication events, SSH logins |
| `/ec2/apache/access` | `/etc/httpd/logs/access_log` | All HTTP/HTTPS requests |
| `/ec2/apache/error` | `/etc/httpd/logs/error_log` | Apache errors and warnings |

### Metric Filter: SSH Failed Logins

Already created in `infrastructure/README.md`. Confirm it exists:

```bash
aws logs describe-metric-filters \
  --log-group-name "/ec2/system/secure" \
  --filter-name-prefix "SSHFailed"
```

### Metric Filter: HTTP 4xx Errors

```bash
aws logs put-metric-filter \
  --log-group-name "/ec2/apache/access" \
  --filter-name "HTTP4xxErrors" \
  --filter-pattern '[ip, id, user, timestamp, request, status_code=4*, size]' \
  --metric-transformations \
    metricName=HTTP4xxCount,metricNamespace=CustomMetrics,metricValue=1
```

### Metric Filter: HTTP 5xx Errors

```bash
aws logs put-metric-filter \
  --log-group-name "/ec2/apache/access" \
  --filter-name "HTTP5xxErrors" \
  --filter-pattern '[ip, id, user, timestamp, request, status_code=5*, size]' \
  --metric-transformations \
    metricName=HTTP5xxCount,metricNamespace=CustomMetrics,metricValue=1
```

### Metric Filter: WordPress Login Failures

WordPress logs failed login attempts to the Apache error log via Wordfence.
Create a filter to count them:

```bash
aws logs put-metric-filter \
  --log-group-name "/ec2/apache/error" \
  --filter-name "WordPressLoginFailures" \
  --filter-pattern '"Wordfence" "blocked" "login"' \
  --metric-transformations \
    metricName=WPLoginFailCount,metricNamespace=CustomMetrics,metricValue=1
```

---

## 3. CloudWatch Alarms Summary

All alarms send notifications to the SNS topic configured in Section 4.
Replace `YOUR_SNS_ARN` and `i-XXXXXXXXXXXXXXXX` throughout.

### CPU Utilization Alarm

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "EC2-HighCPU" \
  --alarm-description "CPU above 80% for 10 minutes" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions YOUR_SNS_ARN \
  --dimensions Name=InstanceId,Value=i-XXXXXXXXXXXXXXXX
```

### Memory Utilization Alarm

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "EC2-HighMemory" \
  --alarm-description "Memory above 85%" \
  --metric-name mem_used_percent \
  --namespace CWAgent \
  --statistic Average \
  --period 300 \
  --threshold 85 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions YOUR_SNS_ARN \
  --dimensions Name=InstanceId,Value=i-XXXXXXXXXXXXXXXX
```

### Disk Utilization Alarm

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "EC2-HighDisk" \
  --alarm-description "Disk above 85% on root volume" \
  --metric-name disk_used_percent \
  --namespace CWAgent \
  --statistic Average \
  --period 300 \
  --threshold 85 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions YOUR_SNS_ARN \
  --dimensions \
    Name=InstanceId,Value=i-XXXXXXXXXXXXXXXX \
    Name=path,Value=/ \
    Name=fstype,Value=xfs
```

### SSH Brute Force Alarm

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "SSH-BruteForce" \
  --alarm-description "More than 10 SSH failures in 5 minutes" \
  --metric-name SSHFailedLoginCount \
  --namespace CustomMetrics \
  --statistic Sum \
  --period 300 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions YOUR_SNS_ARN
```

### HTTP 5xx Error Spike Alarm

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "Apache-5xx-Spike" \
  --alarm-description "More than 20 HTTP 5xx errors in 5 minutes" \
  --metric-name HTTP5xxCount \
  --namespace CustomMetrics \
  --statistic Sum \
  --period 300 \
  --threshold 20 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions YOUR_SNS_ARN
```

### WordPress Login Failure Alarm

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "WP-LoginBruteForce" \
  --alarm-description "More than 15 WordPress login failures in 5 minutes" \
  --metric-name WPLoginFailCount \
  --namespace CustomMetrics \
  --statistic Sum \
  --period 300 \
  --threshold 15 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions YOUR_SNS_ARN
```

### Alarm Summary Table

| Alarm name | Metric | Threshold | Action |
|-----------|--------|-----------|--------|
| EC2-HighCPU | CPUUtilization | > 80% for 10 min | SNS email |
| EC2-HighMemory | mem_used_percent | > 85% | SNS email |
| EC2-HighDisk | disk_used_percent | > 85% | SNS email |
| SSH-BruteForce | SSHFailedLoginCount | > 10 in 5 min | SNS email |
| Apache-5xx-Spike | HTTP5xxCount | > 20 in 5 min | SNS email |
| WP-LoginBruteForce | WPLoginFailCount | > 15 in 5 min | SNS email |

---

## 4. SNS Email Notifications

### Create an SNS Topic

```bash
aws sns create-topic --name lamp-wordpress-alerts
```

Note the `TopicArn` returned — this is your `YOUR_SNS_ARN` value.

### Subscribe Your Email

```bash
aws sns subscribe \
  --topic-arn YOUR_SNS_ARN \
  --protocol email \
  --notification-endpoint your-email@domain.com
```

Check your inbox and click the confirmation link in the subscription
confirmation email from AWS.

### Confirm Subscription is Active

```bash
aws sns list-subscriptions-by-topic --topic-arn YOUR_SNS_ARN
```

Expected: `"SubscriptionArn"` should show a full ARN, not `PendingConfirmation`.

---

## 5. Datadog Agent Setup

Datadog provides richer application-level monitoring than CloudWatch alone,
including per-process metrics, service-level dashboards, and ML-based
anomaly detection.

### Create a Datadog Account

Sign up at [datadoghq.com](https://www.datadoghq.com) — the free tier
covers up to 5 hosts and is sufficient for this project.

### Install the Datadog Agent on Amazon Linux 2023

```bash
DD_API_KEY=YOUR_DATADOG_API_KEY \
DD_SITE="datadoghq.com" \
bash -c "$(curl -L https://s3.amazonaws.com/dd-agent/scripts/install_script_agent7.sh)"
```

Replace `YOUR_DATADOG_API_KEY` with the API key from your Datadog account
under **Organization Settings → API Keys**.

### Verify the Agent is Running

```bash
sudo systemctl status datadog-agent
sudo datadog-agent status
```

The agent status output will list all active checks and any errors.

### Configure the Agent

```bash
sudo nano /etc/datadog-agent/datadog.yaml
```

Ensure the following are set:

```yaml
api_key: YOUR_DATADOG_API_KEY
site: datadoghq.com

# Tag this host for filtering in the Datadog UI
tags:
  - env:production
  - project:lamp-wordpress-security
  - cloud:aws

# Enable log collection
logs_enabled: true

# Enable process monitoring
process_config:
  process_collection:
    enabled: true
```

```bash
sudo systemctl restart datadog-agent
```

---

## 6. Datadog Integrations

### Apache Integration

```bash
sudo cp /etc/datadog-agent/conf.d/apache.d/conf.yaml.example \
        /etc/datadog-agent/conf.d/apache.d/conf.yaml

sudo nano /etc/datadog-agent/conf.d/apache.d/conf.yaml
```

```yaml
init_config:

instances:
  - apache_status_url: http://localhost/server-status?auto
    tags:
      - service:apache
      - env:production
```

Enable `mod_status` in Apache (required for the Datadog Apache check):

```bash
sudo nano /etc/httpd/conf.d/status.conf
```

```apache
<Location "/server-status">
    SetHandler server-status
    Require local
</Location>
```

```bash
sudo systemctl reload httpd
sudo systemctl restart datadog-agent
```

### MySQL Integration

```bash
sudo cp /etc/datadog-agent/conf.d/mysql.d/conf.yaml.example \
        /etc/datadog-agent/conf.d/mysql.d/conf.yaml

sudo nano /etc/datadog-agent/conf.d/mysql.d/conf.yaml
```

```yaml
init_config:

instances:
  - host: 127.0.0.1
    port: 3306
    username: datadog
    password: DATADOG_MYSQL_PASSWORD
    tags:
      - service:mysql
      - env:production
    options:
      replication: false
      galera_cluster: false
      extra_status_metrics: true
      extra_innodb_metrics: true
      extra_performance_metrics: false
      schema_size_metrics: false
      disable_innodb_metrics: false
```

Create a dedicated Datadog MySQL monitoring user with read-only privileges:

```sql
CREATE USER 'datadog'@'localhost' IDENTIFIED BY 'DATADOG_MYSQL_PASSWORD';
GRANT REPLICATION CLIENT ON *.* TO 'datadog'@'localhost';
GRANT PROCESS ON *.* TO 'datadog'@'localhost';
GRANT SELECT ON performance_schema.* TO 'datadog'@'localhost';
FLUSH PRIVILEGES;
```

### Log Collection Integration

```bash
sudo nano /etc/datadog-agent/conf.d/apache.d/conf.yaml
```

Add log collection to the existing config:

```yaml
logs:
  - type: file
    path: /etc/httpd/logs/access_log
    source: apache
    service: wordpress
    tags:
      - env:production

  - type: file
    path: /etc/httpd/logs/error_log
    source: apache
    service: wordpress
    tags:
      - env:production
```

```bash
sudo systemctl restart datadog-agent
```

---

## 7. Datadog Monitors

Create these monitors in the Datadog UI under **Monitors → New Monitor**.

| Monitor | Type | Condition | Alert |
|---------|------|-----------|-------|
| High CPU | Metric | `avg(last_5m):avg:system.cpu.user{host:YOUR_HOST} > 80` | Email |
| High disk | Metric | `avg(last_5m):avg:system.disk.in_use{host:YOUR_HOST} > 0.85` | Email |
| Apache down | Process check | `apache` process not running for 1 min | Email + PagerDuty |
| MySQL down | Process check | `mysqld` process not running for 1 min | Email + PagerDuty |
| Apache error rate anomaly | Anomaly | 3 std deviations above baseline on `apache.net.hits` | Email |
| MySQL slow queries | Metric | `avg(last_15m):avg:mysql.performance.slow_queries > 10` | Email |

---

## 8. Log Review Procedures

Regular manual log review catches patterns that automated alerts miss.

### Check for Failed SSH Logins

```bash
sudo grep "Failed password" /var/log/secure | tail -50
sudo grep "Invalid user" /var/log/secure | tail -50
```

### Check for Successful SSH Logins (Confirm Only Yours)

```bash
sudo grep "Accepted publickey" /var/log/secure | tail -20
```

### Check Apache for Suspicious Requests

```bash
# Top 10 IPs by request volume
sudo awk '{print $1}' /etc/httpd/logs/access_log | sort | uniq -c | sort -rn | head -10

# All 404 errors — look for scanning patterns
sudo grep '" 404 ' /etc/httpd/logs/access_log | tail -30

# All POST requests to non-WordPress endpoints
sudo grep '"POST' /etc/httpd/logs/access_log | grep -v "wp-" | tail -20

# Requests targeting sensitive WordPress files
sudo grep -E "(wp-config|xmlrpc|\.env|\.git)" /etc/httpd/logs/access_log | tail -20
```

### Check fail2ban Bans

```bash
sudo fail2ban-client status sshd
sudo fail2ban-client status apache-auth
sudo fail2ban-client status apache-badbots
```

### CloudWatch Logs Insights Query

In the AWS console, navigate to **CloudWatch → Logs Insights** and run:
```
fields @timestamp, @message
| filter @logStream like /secure/
| filter @message like /Failed password/
| stats count(*) as failures by bin(5m)
| sort @timestamp desc
| limit 50
```

---
---

## 9. Verification Checklist

| Check | How to verify | Expected result |
|-------|---------------|----------------|
| CloudWatch agent running | `systemctl status amazon-cloudwatch-agent` | Active |
| Log groups exist | AWS Console → CloudWatch → Log groups | All 4 groups present |
| Metric filters active | `aws logs describe-metric-filters` | All filters listed |
| All alarms in OK state | CloudWatch → Alarms | Green / OK |
| SNS subscription confirmed | `aws sns list-subscriptions-by-topic` | Not PendingConfirmation |
| Datadog agent running | `sudo datadog-agent status` | All checks passing |
| Apache check active | Datadog → Infrastructure → HOST | Apache metrics visible |
| MySQL check active | Datadog → Infrastructure → HOST | MySQL metrics visible |
| Logs flowing to Datadog | Datadog → Logs | Apache access/error logs appearing |

---

## References

- [AWS CloudWatch Agent Documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html)
- [CloudWatch Logs Insights Query Syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html)
- [Datadog Amazon Linux Installation](https://docs.datadoghq.com/agent/basic_agent_usage/amazonlinux/)
- [Datadog Apache Integration](https://docs.datadoghq.com/integrations/apache/)
- [Datadog MySQL Integration](https://docs.datadoghq.com/integrations/mysql/)

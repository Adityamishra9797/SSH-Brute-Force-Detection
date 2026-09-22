# SSH-Brute-Force-Detection
AWS Cloud Watch pipeline that detects SSH brute-force attempts

SSH Brute-Force Detection Pipeline (AWS CloudWatch)

Overview
A cloud-native intrusion detection pipeline that monitors EC2 instances for repeated failed SSH login attempts (a classic brute-force indicator) and sends a real-time email alert when a threshold is exceeded.
This project demonstrates the detection side of security operations — log aggregation, writing detection logic, and alerting — mirroring day-to-day work of a SOC analyst or cloud security engineer.

Architecture
EC2 Instances (x3, Amazon Linux 2023)
   │ (rsyslog writes /var/log/secure → CloudWatch Agent tails it)
   ▼
CloudWatch Log Group "ssh-auth-logs" (centralized, one stream per instance)
   │ (Metric Filter matches "Connection closed by authenticating user")
   ▼
CloudWatch Metric: SSHMonitoring / FailedSSHAttempts
   │ (Alarm: Sum > 5 in a 5-minute period)
   ▼
SNS Topic "ssh-brute-force-alert" → Email Notification

Skills Used
AWS EC2 · IAM Roles · CloudWatch Agent · CloudWatch Logs · CloudWatch Metric Filters & Alarms · SNS · rsyslog · Linux systemd/journald

Environment
Region: us-east-1
3x t3.micro Amazon Linux 2023 instances (Free Tier / AWS credits)
Security group: SSH restricted to a single known IP
Build Steps

1. Launch EC2 instances
Launched 3 t3.micro Amazon Linux 2023 instances in the same VPC/subnet to simulate a small fleet. Used "Launch more like this" to keep configuration consistent across instances. Hit the default account vCPU limit (4) partway through — each t3.micro uses 2 vCPUs, so only 2 instances could launch at once; the 3rd was added after a retry.

2. Secure access
Created an RSA key pair (.pem, for OpenSSH/Mac compatibility)
Locked the security group's SSH rule to "My IP" instead of Anywhere (0.0.0.0/0)

3. IAM role
Created EC2-CloudWatch-Role with the CloudWatchAgentServerPolicy managed policy, attached to all 3 instances so the agent can push logs without hardcoded credentials.

4. Install & configure the CloudWatch Agent
sudo yum install amazon-cloudwatch-agent -y
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
Configured to monitor /var/log/secure, forwarding to log group ssh-auth-logs (Standard class, 7-day retention). Started the agent with:
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -s \
  -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json

5. Create a Metric Filter
Filter pattern: "Connection closed by authenticating user" — matches failed SSH auth attempts. Converted into custom metric FailedSSHAttempts (namespace SSHMonitoring), value 1 per match.

6. Create an Alarm + SNS Notification
Metric: SSHMonitoring / FailedSSHAttempts, Statistic: Sum, Period: 5 minutes
Condition: Greater than 5
Notification: new SNS topic ssh-brute-force-alert, subscribed via email, confirmed the subscription link

7. Test the detection
From a separate machine (own laptop, different network context than the instances), deliberately triggered 6+ failed SSH connection attempts within a 5-minute window. Confirmed the alarm transitioned to "In alarm" and the email notification arrived.

What It Teaches
Log aggregation, writing detection logic, and alerting — the monitoring/detection side of security operations. Strong resume talking point: "Built a brute-force detection pipeline using AWS CloudWatch."

Build Notes / Troubleshooting Log
Real issues hit during the build — kept here because working through them is arguably more instructive than a clean walkthrough:
/var/log/secure doesn't exist by default on Amazon Linux 2023. Unlike AL2, AL2023 uses systemd-journald for auth logging (journalctl -u sshd), not a flat file. Had to install and enable rsyslog to get a traditional /var/log/secure file that the CloudWatch Agent could tail.
CloudWatch Agent couldn't read the log file once it existed. /var/log/secure is created -rw------- (root-only). The agent runs as the cwagent user, so it hit silent "permission denied" errors on every read attempt. Fixed by adding cwagent to the root group and setting the file to 640.
Filter pattern mismatch. Because these instances only allow key-based SSH auth (no password auth), failed logins never produce a "Failed password" line — instead sshd logs "Connection closed by authenticating user ... [preauth]". Adjusted the metric filter pattern to match the actual log format instead of the commonly-documented one.
SNS email subscription was pending, not confirmed, until manually resending the confirmation and checking spam — the alarm silently wouldn't have notified anyone until this was caught.
vCPU limit on a fresh AWS account blocked launching all 3 instances in a single batch; had to launch 2, then 1 more.

Cleanup
Terminate EC2 instances, delete the SNS topic and log group after testing to avoid ongoing charges. (IAM role can be left — no cost, reusable for future builds.)

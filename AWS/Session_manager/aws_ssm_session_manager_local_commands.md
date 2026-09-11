# AWS SSM Session Manager — Local Command Reference

## 1. Prerequisites

Install and configure:

- AWS CLI
- AWS Session Manager plugin
- Valid AWS credentials/profile
- IAM permission: `ssm:StartSession`
- Target EC2 instance managed by AWS Systems Manager (SSM)

Verify installation:

```bash
aws --version
session-manager-plugin --version
```

Check the active AWS identity:

```bash
aws sts get-caller-identity
```

List configured profiles:

```bash
aws configure list-profiles
```

Check a specific profile:

```bash
aws configure list --profile <profile-name>
aws sts get-caller-identity --profile <profile-name>
```

---

## 2. Find EC2 Instances

List instances:

```bash
aws ec2 describe-instances
```

Using a profile:

```bash
aws ec2 describe-instances --profile <profile-name>
```

List only running instances:

```bash
aws ec2 describe-instances \
  --filters Name=instance-state-name,Values=running
```

Get instance IDs and private IPs:

```bash
aws ec2 describe-instances \
  --query 'Reservations[].Instances[].[InstanceId,PrivateIpAddress,State.Name]' \
  --output table
```

Filter by Name tag:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=<instance-name>"
```

---

## 3. Check SSM Managed Instances

List SSM managed instances:

```bash
aws ssm describe-instance-information
```

Show useful fields:

```bash
aws ssm describe-instance-information \
  --query 'InstanceInformationList[].[InstanceId,PingStatus,PlatformName,PlatformVersion,AgentVersion]' \
  --output table
```

Filter for a specific instance:

```bash
aws ssm describe-instance-information \
  --filters "Key=InstanceIds,Values=<instance-id>"
```

A healthy instance normally shows:

```text
PingStatus = Online
```

---

## 4. Start an Interactive SSM Session

Basic command:

```bash
aws ssm start-session --target <instance-id>
```

With a specific profile:

```bash
aws ssm start-session \
  --target <instance-id> \
  --profile <profile-name>
```

With a specific region:

```bash
aws ssm start-session \
  --target <instance-id> \
  --region <region> \
  --profile <profile-name>
```

Example:

```bash
aws ssm start-session \
  --target i-0123456789abcdef0 \
  --region ap-south-1 \
  --profile dev
```

Exit the session:

```bash
exit
```

---

## 5. Start Session Using AWS CLI Environment Variables

Set profile:

```bash
export AWS_PROFILE=dev
```

Set region:

```bash
export AWS_REGION=ap-south-1
```

Then:

```bash
aws sts get-caller-identity
aws ssm describe-instance-information
aws ssm start-session --target <instance-id>
```

For Windows PowerShell:

```powershell
$env:AWS_PROFILE="dev"
$env:AWS_REGION="ap-south-1"
```

---

## 6. Run a Single Command Without Interactive Login

SSM `send-command` is useful when you only need to execute a command remotely.

Run Linux command:

```bash
aws ssm send-command \
  --instance-ids <instance-id> \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["uname -a"]'
```

Run multiple commands:

```bash
aws ssm send-command \
  --instance-ids <instance-id> \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["uptime","df -h","free -m"]'
```

Check command status:

```bash
aws ssm list-command-invocations \
  --command-id <command-id> \
  --details
```

Get only status:

```bash
aws ssm list-command-invocations \
  --command-id <command-id> \
  --query 'CommandInvocations[].Status'
```

---

## 7. SSM Run Command With Output

Send command:

```bash
COMMAND_ID=$(aws ssm send-command \
  --instance-ids <instance-id> \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["hostname","uptime","df -h"]' \
  --query 'Command.CommandId' \
  --output text)
```

Check result:

```bash
aws ssm list-command-invocations \
  --command-id "$COMMAND_ID" \
  --details
```

Get stdout:

```bash
aws ssm list-command-invocations \
  --command-id "$COMMAND_ID" \
  --details \
  --query 'CommandInvocations[].CommandPlugins[].Output'
```

---

## 8. Useful Commands After Connecting to EC2

### Identity

```bash
whoami
id
hostname
hostname -f
```

### OS information

```bash
uname -a
cat /etc/os-release
```

### CPU

```bash
lscpu
nproc
uptime
top
```

### Memory

```bash
free -h
vmstat 1 5
```

### Disk

```bash
df -h
df -i
lsblk
du -sh *
```

Find large files:

```bash
sudo du -ah /var | sort -rh | head -20
```

### Processes

```bash
ps aux
ps -ef
top
```

Find a process:

```bash
ps -ef | grep java
```

Better:

```bash
pgrep -af java
```

### Network

```bash
ip addr
ip route
ss -lntp
ss -antp
```

Check DNS:

```bash
cat /etc/resolv.conf
getent hosts example.com
```

Test connectivity:

```bash
ping <host>
curl -I https://example.com
```

Test a TCP port:

```bash
nc -vz <host> <port>
```

---

## 9. Java / Spring Boot Production Checks

Check Java version:

```bash
java -version
```

Find Java process:

```bash
pgrep -af java
```

Find process details:

```bash
ps -ef | grep java
```

Check listening ports:

```bash
sudo ss -lntp | grep java
```

Check application logs:

```bash
tail -f /path/to/application.log
```

Search errors:

```bash
grep -i "error" /path/to/application.log
```

Search exceptions:

```bash
grep -i "exception" /path/to/application.log
```

Search recent errors:

```bash
grep -iE "error|exception|failed|timeout" /path/to/application.log | tail -100
```

Check Spring Boot process memory:

```bash
ps -o pid,ppid,%cpu,%mem,rss,vsz,cmd -p <pid>
```

---

## 10. systemd Service Troubleshooting

List services:

```bash
systemctl list-units --type=service
```

Check service:

```bash
sudo systemctl status <service-name>
```

Restart:

```bash
sudo systemctl restart <service-name>
```

Start:

```bash
sudo systemctl start <service-name>
```

Stop:

```bash
sudo systemctl stop <service-name>
```

Enable at boot:

```bash
sudo systemctl enable <service-name>
```

View logs:

```bash
sudo journalctl -u <service-name>
```

Follow logs:

```bash
sudo journalctl -u <service-name> -f
```

Recent logs:

```bash
sudo journalctl -u <service-name> --since "30 minutes ago"
```

---

## 11. Check SSM Agent on EC2

Amazon Linux / RHEL / CentOS:

```bash
sudo systemctl status amazon-ssm-agent
```

Follow SSM Agent logs:

```bash
sudo journalctl -u amazon-ssm-agent -f
```

Common log location:

```bash
sudo tail -f /var/log/amazon/ssm/amazon-ssm-agent.log
```

Check agent version:

```bash
amazon-ssm-agent --version
```

Restart agent:

```bash
sudo systemctl restart amazon-ssm-agent
```

---

## 12. Port Forwarding Through SSM

SSM can create a secure tunnel without opening an inbound security-group port.

Basic port forwarding:

```bash
aws ssm start-session \
  --target <instance-id> \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["8080"],"localPortNumber":["8080"]}'
```

Example for PostgreSQL:

```bash
aws ssm start-session \
  --target <instance-id> \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["<rds-endpoint>"],"portNumber":["5432"],"localPortNumber":["5432"]}'
```

Then connect locally:

```bash
psql -h localhost -p 5432 -U <username> -d <database>
```

This is useful for production troubleshooting when direct access to the RDS endpoint is not allowed from the developer laptop.

---

## 13. Useful AWS CLI SSM Commands

List SSM documents:

```bash
aws ssm list-documents
```

Describe a document:

```bash
aws ssm describe-document \
  --name "AWS-RunShellScript"
```

List sessions:

```bash
aws ssm describe-sessions \
  --state Active
```

Terminate a session:

```bash
aws ssm terminate-session \
  --session-id <session-id>
```

Get SSM parameters:

```bash
aws ssm get-parameter \
  --name <parameter-name>
```

Get encrypted SecureString parameter:

```bash
aws ssm get-parameter \
  --name <parameter-name> \
  --with-decryption
```

---

## 14. SSM Troubleshooting Checklist

### Instance does not appear in SSM

Check:

```bash
aws ssm describe-instance-information
```

Then verify:

1. EC2 instance is running.
2. SSM Agent is installed and running.
3. EC2 instance has an IAM role/profile containing suitable SSM permissions.
4. Instance can reach SSM endpoints.
5. Region is correct.
6. AWS CLI credentials have `ssm:StartSession`.

Check agent:

```bash
sudo systemctl status amazon-ssm-agent
```

Check agent logs:

```bash
sudo journalctl -u amazon-ssm-agent --since "30 minutes ago"
```

---

## 15. Common SSM Errors

### TargetNotConnected

Typical causes:

- SSM Agent is offline.
- IAM instance role is missing/incorrect.
- Network path to SSM endpoints is unavailable.
- Wrong AWS region.
- Instance is not registered with SSM.

Check:

```bash
aws ssm describe-instance-information \
  --query 'InstanceInformationList[].[InstanceId,PingStatus,PlatformName,AgentVersion]' \
  --output table
```

---

### AccessDeniedException

Check current identity:

```bash
aws sts get-caller-identity
```

Verify the caller has permission for:

```text
ssm:StartSession
ssm:DescribeInstanceInformation
ssm:TerminateSession
```

For port forwarding, the caller also needs permission to start the relevant SSM document/session.

---

### Session Manager Plugin Not Found

Check:

```bash
session-manager-plugin --version
```

If not found, install the Session Manager plugin for your operating system and verify again.

---

## 16. Production-Safe Investigation Commands

Prefer read-only commands first:

```bash
hostname
uptime
free -h
df -h
df -i
ps -ef
ss -lntp
systemctl status <service>
journalctl -u <service> --since "15 minutes ago"
tail -100 /path/to/application.log
```

Avoid immediately running:

```bash
rm -rf
kill -9
systemctl restart
reboot
```

First collect evidence, identify the impact, and confirm the remediation.

---

## 17. Recommended Production Workflow

```text
Laptop
  |
  | AWS CLI + Session Manager
  v
AWS Systems Manager
  |
  v
EC2 Instance
  |
  +--> Application logs
  +--> Java process
  +--> OS metrics
  +--> Network checks
  +--> systemd
```

Typical investigation:

```bash
aws sts get-caller-identity

aws ssm describe-instance-information \
  --query 'InstanceInformationList[].[InstanceId,PingStatus]' \
  --output table

aws ssm start-session --target <instance-id>

hostname
uptime
free -h
df -h
ps -ef | grep java
ss -lntp

sudo journalctl -u <service> --since "30 minutes ago"
tail -100 /path/to/application.log
```

---

## 18. Useful Aliases

Add to `~/.bashrc`:

```bash
alias awswho='aws sts get-caller-identity'
alias ssm-list='aws ssm describe-instance-information'
alias ec2-running='aws ec2 describe-instances --filters Name=instance-state-name,Values=running'
```

Example:

```bash
awswho
ssm-list
ec2-running
```

For a frequently used instance:

```bash
alias ssm-prod='aws ssm start-session --target <instance-id> --region ap-south-1 --profile prod'
```

---

## 19. Senior DevOps Production Notes

### Session Manager vs SSH

| Area | SSM Session Manager | SSH |
|---|---|---|
| Inbound port 22 | Not required | Usually required |
| Bastion host | Often unnecessary | Commonly used |
| IAM integration | Strong | Limited |
| Centralized access control | Yes | Requires additional setup |
| Auditability | CloudTrail/session logging options | Depends on SSH/audit setup |
| Key management | No SSH key required | SSH keys required |
| Private subnet access | Supported with correct SSM networking | Requires network path/bastion |
| Recommended AWS production access | Usually preferred | Use when specifically required |

### Production recommendation

For AWS EC2 workloads, prefer:

```text
Developer
   |
   v
AWS IAM / Identity Center
   |
   v
SSM Session Manager
   |
   v
Private EC2
```

rather than:

```text
Internet
   |
   v
Bastion
   |
   v
SSH
   |
   v
EC2
```

Use least-privilege IAM, session logging where appropriate, and avoid exposing SSH to the public internet unless there is a justified requirement.

---

## 20. Quick Command Cheat Sheet

```bash
# AWS identity
aws sts get-caller-identity

# Profiles
aws configure list-profiles

# EC2
aws ec2 describe-instances

# SSM managed instances
aws ssm describe-instance-information

# Start session
aws ssm start-session --target <instance-id>

# Active sessions
aws ssm describe-sessions --state Active

# Terminate session
aws ssm terminate-session --session-id <session-id>

# Remote command
aws ssm send-command \
  --instance-ids <instance-id> \
  --document-name AWS-RunShellScript \
  --parameters 'commands=["uptime"]'

# Port forwarding
aws ssm start-session \
  --target <instance-id> \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["8080"],"localPortNumber":["8080"]}'

# EC2 health
uptime
free -h
df -h
df -i

# Processes
ps -ef
pgrep -af java

# Network
ip addr
ip route
ss -lntp
curl -I https://example.com

# Logs
journalctl -u <service> --since "30 minutes ago"
tail -f /path/to/application.log

# SSM agent
systemctl status amazon-ssm-agent
journalctl -u amazon-ssm-agent -f
```

## Production Principle

**SSM Session Manager should be treated as an operational access mechanism, not a replacement for proper observability.**

For production incidents, use:

**CloudWatch/Prometheus/Grafana → identify the problem → SSM → validate on the host → collect evidence → fix → verify → document RCA.**

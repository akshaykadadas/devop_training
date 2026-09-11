# AWS Systems Manager (SSM) — Related Production Topics with Examples

This guide extends the AWS SSM Session Manager command reference into practical production DevOps topics.

---

# 1. SSM Architecture

## What

AWS Systems Manager allows you to manage EC2 instances without depending on traditional SSH access.

## Architecture

```text
Developer Laptop
      |
      | AWS CLI
      v
AWS IAM / Identity Center
      |
      v
SSM Session Manager
      |
      v
SSM Agent
      |
      v
EC2 Instance
```

The SSM Agent runs on the EC2 instance and communicates with AWS Systems Manager.

## Production benefit

A private EC2 instance can be accessed without exposing port 22 to the internet.

---

# 2. IAM for Session Manager

## What

IAM controls who can start, monitor, and terminate SSM sessions.

A user/role commonly needs permissions such as:

```json
{
  "Effect": "Allow",
  "Action": [
    "ssm:StartSession",
    "ssm:DescribeInstanceInformation",
    "ssm:TerminateSession"
  ],
  "Resource": "*"
}
```

In production, replace `"Resource": "*"` with appropriately scoped resources where practical.

## Example

Developer:

```bash
aws sts get-caller-identity
```

Then:

```bash
aws ssm start-session --target i-0123456789abcdef0
```

If the IAM policy does not allow the operation, the user may receive:

```text
AccessDeniedException
```

## Production recommendation

Use separate access levels:

```text
Developer
  -> Read-only / limited SSM access

Support Engineer
  -> SSM + troubleshooting commands

Production Admin
  -> Elevated access with approval
```

---

# 3. EC2 Instance Profile for SSM

The EC2 instance itself needs an IAM role that allows the SSM Agent to communicate with Systems Manager.

A common AWS-managed policy is:

```text
AmazonSSMManagedInstanceCore
```

Conceptually:

```text
EC2
 |
 | IAM Instance Role
 v
SSM Agent
 |
 v
AWS Systems Manager
```

## Verify from AWS CLI

```bash
aws ssm describe-instance-information \
  --query 'InstanceInformationList[].[InstanceId,PingStatus,AgentVersion]' \
  --output table
```

Expected:

```text
i-0123456789abcdef0    Online    3.x.x
```

---

# 4. SSM Agent

## What

SSM Agent is the component running on the EC2 instance that enables Systems Manager operations.

Check:

```bash
sudo systemctl status amazon-ssm-agent
```

Check logs:

```bash
sudo journalctl -u amazon-ssm-agent --since "30 minutes ago"
```

Restart:

```bash
sudo systemctl restart amazon-ssm-agent
```

Check version:

```bash
amazon-ssm-agent --version
```

## Production incident example

### Symptoms

Instance is running but:

```bash
aws ssm start-session --target i-0123456789abcdef0
```

returns:

```text
TargetNotConnected
```

### Investigation

```bash
aws ssm describe-instance-information
```

If the instance is missing or offline, check the EC2 host:

```bash
sudo systemctl status amazon-ssm-agent
```

Then:

```bash
sudo journalctl -u amazon-ssm-agent --since "15 minutes ago"
```

### Possible root causes

- SSM Agent stopped
- IAM role missing
- Network connectivity problem
- Wrong region
- Instance registration problem

---

# 5. SSM and Private Subnet

SSM is particularly useful for EC2 instances in private subnets.

## Architecture

```text
                 AWS Cloud
                     |
             +-------+-------+
             |               |
          SSM Service     CloudWatch
             ^
             |
       SSM Agent
             |
       Private EC2
             |
        Private Subnet
```

The instance does not need a public IP just to use SSM.

## Network considerations

Depending on the architecture, private instances can communicate with Systems Manager through:

- NAT Gateway
- Appropriate VPC endpoints

Common interface endpoints include:

```text
com.amazonaws.<region>.ssm
com.amazonaws.<region>.ssmmessages
com.amazonaws.<region>.ec2messages
```

Example for Mumbai:

```text
com.amazonaws.ap-south-1.ssm
com.amazonaws.ap-south-1.ssmmessages
com.amazonaws.ap-south-1.ec2messages
```

---

# 6. SSM Session Manager vs SSH

## Traditional architecture

```text
Internet
   |
   v
Bastion
   |
   | SSH :22
   v
Private EC2
```

## SSM architecture

```text
Developer
   |
   v
IAM
   |
   v
SSM
   |
   v
Private EC2
```

## Why SSM is usually preferred

- No inbound SSH port required
- IAM-based access
- Easier centralized access control
- Works well with private subnets
- Session logging can be configured
- No SSH key distribution required

---

# 7. Session Logging and Auditing

For production, investigate who accessed which instance and when.

AWS CloudTrail records AWS API activity.

Example:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=StartSession
```

You can also configure Session Manager preferences to send session logs to services such as:

```text
CloudWatch Logs
S3
```

## Production architecture

```text
Developer
    |
    v
SSM Session
    |
    v
EC2

Session Logs
    |
    +----> CloudWatch Logs
    |
    +----> S3
```

This improves auditability during production investigations.

---

# 8. SSM Run Command

## What

`send-command` executes commands remotely without opening an interactive shell.

Example:

```bash
aws ssm send-command \
  --instance-ids i-0123456789abcdef0 \
  --document-name AWS-RunShellScript \
  --parameters 'commands=["uptime","df -h","free -h"]'
```

Get command ID:

```bash
COMMAND_ID=$(aws ssm send-command \
  --instance-ids i-0123456789abcdef0 \
  --document-name AWS-RunShellScript \
  --parameters 'commands=["uptime"]' \
  --query 'Command.CommandId' \
  --output text)
```

Check result:

```bash
aws ssm list-command-invocations \
  --command-id "$COMMAND_ID" \
  --details
```

## Production use case

You have 20 EC2 instances and want to check disk usage.

Instead of logging into each server:

```text
SSM Run Command
      |
      +--> EC2-1
      +--> EC2-2
      +--> EC2-3
      ...
      +--> EC2-20
```

---

# 9. SSM Documents

SSM Documents define actions that Systems Manager can execute.

Common document:

```text
AWS-RunShellScript
```

Example:

```bash
aws ssm send-command \
  --instance-ids <instance-id> \
  --document-name AWS-RunShellScript \
  --parameters 'commands=["hostname"]'
```

Other useful documents can support:

- PowerShell
- Port forwarding
- Automation
- Patch operations

List documents:

```bash
aws ssm list-documents
```

Describe one:

```bash
aws ssm describe-document \
  --name AWS-RunShellScript
```

---

# 10. Port Forwarding

One of the most useful production troubleshooting capabilities.

## Scenario

RDS is private:

```text
Developer Laptop
      X
      |
      | No direct access
      |
Private RDS
```

Instead:

```text
Developer Laptop
      |
      v
SSM Session
      |
      v
EC2
      |
      v
Private RDS
```

Start tunnel:

```bash
aws ssm start-session \
  --target <ec2-instance-id> \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["<rds-endpoint>"],"portNumber":["5432"],"localPortNumber":["5432"]}'
```

Then:

```bash
psql \
  -h localhost \
  -p 5432 \
  -U <username> \
  -d <database>
```

## Important

The local port:

```text
localhost:5432
```

is forwarded through the EC2 instance to:

```text
RDS:5432
```

---

# 11. Port Forwarding to Spring Boot

Suppose the application exposes:

```text
EC2:8080
```

Start:

```bash
aws ssm start-session \
  --target <instance-id> \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["8080"],"localPortNumber":["8080"]}'
```

Then locally:

```bash
curl http://localhost:8080/actuator/health
```

Traffic path:

```text
localhost:8080
      |
      v
SSM tunnel
      |
      v
EC2:8080
      |
      v
Spring Boot
```

This is useful for troubleshooting private services without exposing the application publicly.

---

# 12. SSM Parameter Store

Parameter Store stores configuration values and secrets.

Example:

```bash
aws ssm get-parameter \
  --name /myapp/dev/database/url
```

Encrypted parameter:

```bash
aws ssm get-parameter \
  --name /myapp/prod/database/password \
  --with-decryption
```

Example hierarchy:

```text
/myapp/dev/database/url
/myapp/dev/database/username
/myapp/dev/database/password

/myapp/prod/database/url
/myapp/prod/database/username
/myapp/prod/database/password
```

## Spring Boot concept

```text
Spring Boot
    |
    v
AWS Parameter Store
    |
    +--> DB URL
    +--> DB username
    +--> configuration
```

For highly sensitive secrets, evaluate AWS Secrets Manager when secret rotation and lifecycle management are required.

---

# 13. SSM Automation

Automation runs predefined operational workflows.

Example conceptual workflow:

```text
Detect unhealthy EC2
       |
       v
Run diagnostics
       |
       v
Collect logs
       |
       v
Restart service
       |
       v
Validate health
```

This is useful for repeatable operational procedures.

---

# 14. SSM Patch Manager

Patch Manager can help manage OS patching.

Production architecture:

```text
EC2 Fleet
   |
   v
SSM
   |
   v
Patch Baseline
   |
   v
Scheduled Maintenance
```

Before patching production:

1. Validate backup/recovery.
2. Test in lower environment.
3. Define maintenance window.
4. Patch a small group.
5. Validate application health.
6. Continue rollout.

Avoid blindly patching the entire fleet simultaneously.

---

# 15. Maintenance Windows

Maintenance Windows define when operational tasks can run.

Example:

```text
Sunday
02:00 - 04:00 IST
```

Possible tasks:

```text
OS patching
Application restart
Log cleanup
Health checks
Automation
```

Production principle:

> Maintenance should be scheduled, observable, reversible, and tested.

---

# 16. SSM Inventory

SSM Inventory collects information about managed instances.

Useful information includes:

```text
OS
Installed software
Applications
Network information
Instance metadata
```

This is useful when managing a large EC2 fleet.

Example problem:

> Which production instances have an outdated Java version?

Inventory can help identify affected machines before remediation.

---

# 17. Production Incident: Disk Full

## Symptoms

Application becomes unstable.

Logs may show:

```text
No space left on device
```

## Investigation

Start session:

```bash
aws ssm start-session --target <instance-id>
```

Check:

```bash
df -h
```

Check inode usage:

```bash
df -i
```

Find large directories:

```bash
sudo du -xhd1 /var | sort -h
```

Find large files:

```bash
sudo find /var -type f -size +500M -ls
```

Check logs:

```bash
sudo journalctl --disk-usage
```

## Root cause

Example:

```text
Application logs were not rotated.
```

## Fix

Implement proper log rotation and retention.

Do not blindly delete active application files.

## Prevention

- Log rotation
- Disk usage alert
- CloudWatch monitoring
- Retention policy
- Automated cleanup where safe

---

# 18. Production Incident: Java CPU 100%

## Symptoms

ALB latency increases.

CloudWatch shows:

```text
CPUUtilization = 99%
```

## Investigation

```bash
top
```

Find Java process:

```bash
pgrep -af java
```

Check CPU:

```bash
ps -eo pid,ppid,%cpu,%mem,cmd --sort=-%cpu | head
```

Identify threads:

```bash
top -H -p <java-pid>
```

Convert the high CPU thread ID to hexadecimal:

```bash
printf '%x\n' <thread-id>
```

Then correlate with a Java thread dump.

## Root cause examples

- Infinite loop
- Excessive GC
- High-volume request
- Bad query processing
- Thread contention
- Unexpected traffic

## Prevention

Use:

```text
CloudWatch
+
Application metrics
+
Prometheus
+
Grafana
+
Java/JVM metrics
```

---

# 19. Production Incident: Application Port Not Listening

## Symptoms

ALB health check fails.

Check:

```bash
ss -lntp
```

Expected:

```text
LISTEN ... :8080 ... java
```

If missing:

```bash
ps -ef | grep java
```

Check service:

```bash
sudo systemctl status <service>
```

Check logs:

```bash
sudo journalctl -u <service> --since "20 minutes ago"
```

## Possible root causes

- Spring Boot failed during startup
- Port configuration changed
- Application crashed
- Dependency connection failure
- Configuration/secret missing

---

# 20. Production Incident: Cannot Connect to RDS

## Investigation

From EC2:

```bash
getent hosts <rds-endpoint>
```

Test TCP:

```bash
nc -vz <rds-endpoint> 5432
```

Check route:

```bash
ip route
```

Check application logs:

```bash
grep -iE "connection|timeout|postgres|database" /path/to/application.log | tail -100
```

## Investigation layers

```text
DNS
 |
v
Route
 |
v
Security Group
 |
v
Network ACL
 |
v
RDS availability
 |
v
Database credentials
 |
v
Connection pool
```

Do not assume every database connection problem is a security-group issue.

---

# 21. SSM + CloudWatch + Prometheus

SSM is for operational access.

CloudWatch is for AWS infrastructure/service monitoring.

Prometheus/Grafana is useful for application and Kubernetes metrics.

Recommended model:

```text
                 Production
                     |
       +-------------+-------------+
       |             |             |
   CloudWatch    Prometheus      Logs
       |             |             |
       +-------------+-------------+
                     |
                  Alert
                     |
                     v
              Engineer
                     |
                     v
               SSM Session
                     |
                     v
                 EC2/EKS
```

Principle:

> Observe first, connect second.

---

# 22. SSM With EKS

For EKS, do not treat SSM as the primary way to troubleshoot Kubernetes workloads.

Use Kubernetes commands first:

```bash
kubectl get pods
kubectl describe pod <pod>
kubectl logs <pod>
kubectl get events
```

For EC2-backed worker-node investigation, SSM can provide host-level access when needed.

Architecture:

```text
Developer
   |
   +--> kubectl
   |      |
   |      v
   |     EKS
   |
   +--> SSM
          |
          v
       Worker Node
```

Use SSM for node/OS-level problems, not as a replacement for Kubernetes observability.

---

# 23. Recommended Security Model

Production access should look like:

```text
Developer
   |
   v
IAM Identity Center / IAM
   |
   v
Least Privilege
   |
   v
SSM Session Manager
   |
   v
Private EC2
```

Avoid:

```text
0.0.0.0/0 -> TCP 22 -> EC2
```

unless there is a specific justified requirement.

Additional controls:

- MFA
- Short-lived credentials
- Least privilege
- Session logging
- CloudTrail
- Approval workflow for sensitive production access
- Separate production roles
- Regular IAM review

---

# 24. Senior-Level Interview Questions

## Q1. Why use SSM instead of SSH?

Answer:

SSM provides IAM-controlled access without requiring inbound SSH connectivity. It is especially useful for private EC2 instances and reduces SSH key/bastion management.

---

## Q2. Can SSM work without a public IP?

Yes.

The instance needs a network path to the required AWS Systems Manager endpoints, typically through NAT or suitable VPC endpoints.

---

## Q3. What happens internally when you run `aws ssm start-session`?

Conceptually:

```text
AWS CLI
   |
   v
SSM StartSession API
   |
   v
Systems Manager
   |
   v
SSM Agent on EC2
   |
   v
Interactive session
```

The SSM Agent maintains communication with Systems Manager.

---

## Q4. What is the difference between Run Command and Session Manager?

```text
Run Command
    -> Execute predefined/non-interactive commands

Session Manager
    -> Interactive shell/session
```

Example:

```text
Check 50 servers -> Run Command
Debug one server -> Session Manager
```

---

## Q5. Why can an EC2 instance be healthy but unavailable in SSM?

Possible reasons:

- SSM Agent stopped
- IAM role problem
- Network endpoint problem
- Wrong region
- SSM registration issue

---

## Q6. How would you troubleshoot `TargetNotConnected`?

Use:

```bash
aws ssm describe-instance-information
```

Then on EC2:

```bash
sudo systemctl status amazon-ssm-agent
sudo journalctl -u amazon-ssm-agent --since "30 minutes ago"
```

Then verify:

```text
IAM role
Network
Region
SSM Agent
SSM endpoints
```

---

# 25. Production Troubleshooting Framework

Use this sequence during incidents:

```text
1. Alert
   |
2. Confirm impact
   |
3. Check CloudWatch / Grafana
   |
4. Identify affected instance/service
   |
5. Start SSM session if host investigation is needed
   |
6. Collect logs
   |
7. Check CPU / memory / disk / network
   |
8. Form hypothesis
   |
9. Validate hypothesis
   |
10. Apply controlled fix
   |
11. Verify recovery
   |
12. Document RCA
   |
13. Add prevention
```

---

# 26. Practical Hands-On Lab

## Lab 1 — Connect to EC2

```bash
aws sts get-caller-identity

aws ssm describe-instance-information \
  --output table

aws ssm start-session \
  --target <instance-id>
```

Inside EC2:

```bash
hostname
uptime
free -h
df -h
ss -lntp
```

---

## Lab 2 — Run Remote Diagnostics

```bash
aws ssm send-command \
  --instance-ids <instance-id> \
  --document-name AWS-RunShellScript \
  --parameters 'commands=["hostname","uptime","free -h","df -h"]'
```

Retrieve the command output.

---

## Lab 3 — Port Forward to PostgreSQL

Create an SSM tunnel:

```bash
aws ssm start-session \
  --target <instance-id> \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["<rds-endpoint>"],"portNumber":["5432"],"localPortNumber":["5432"]}'
```

Connect:

```bash
psql -h localhost -p 5432 -U <username> -d <database>
```

---

## Lab 4 — Production Incident Simulation

Simulate:

```text
Disk usage > 90%
```

Investigate:

```bash
df -h
sudo du -xhd1 /var | sort -h
```

Determine:

```text
Symptoms
Investigation
Evidence
Root Cause
Fix
Prevention
```

---

# 27. Key Commands to Memorize

```bash
aws sts get-caller-identity

aws ssm describe-instance-information

aws ssm start-session --target <instance-id>

aws ssm describe-sessions --state Active

aws ssm terminate-session --session-id <session-id>

aws ssm send-command \
  --instance-ids <instance-id> \
  --document-name AWS-RunShellScript \
  --parameters 'commands=["uptime"]'

sudo systemctl status amazon-ssm-agent

sudo journalctl -u amazon-ssm-agent

df -h
free -h
uptime
ps -ef
ss -lntp
journalctl
tail -f
```

---

# 28. Learning Path From SSM to Production AWS DevOps

Recommended progression:

```text
Linux
  |
Networking
  |
IAM
  |
EC2
  |
SSM Session Manager
  |
CloudWatch
  |
ALB
  |
S3 / RDS / EFS
  |
Docker
  |
ECR
  |
Kubernetes
  |
EKS
  |
Terraform
  |
GitHub Actions
  |
DevSecOps
  |
Prometheus / Grafana
  |
Production Troubleshooting
  |
Deployment Strategies
  |
System Design
```

The important goal is not memorizing SSM commands.

The goal is to understand:

**Alert → Investigation → Evidence → Root Cause → Remediation → Verification → Prevention**
